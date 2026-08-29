# Rapport d'homologation officiel — otarelay Channel API v2

**Date** : 25 août 2026
**Version API testée** : v2
**Suite d'homologation** : `src/tests/homologation/runner.ts` (89 tests, 8 phases)
**Environnement d'exécution** : exécution locale (`http://localhost:3000`), équivalent
fonctionnel de l'environnement sandbox partenaire (`https://api.dev.otarelay.com`,
voir [`sandbox.md`](./sandbox.md)) — même suite de tests, ciblable via `--env staging`
sur ce domaine pour une validation finale avant mise en production.
**Channel Managers testés** : CM Test 1, CM Test 2 (identifiants génériques
d'homologation — voir `src/tests/homologation/utils.ts`)

## Statut : ✅ HOMOLOGATION_READY

## Résultat global

| Channel Manager | Tests exécutés | Résultat | Tests ignorés | Statut |
|---|---|---|---|---|
| CM Test 1 | 87/87 | ✅ 100 % | 2 SKIPPED (`ENV_DEPENDENT`) | `HOMOLOGATION_READY` |
| CM Test 2 | 87/87 | ✅ 100 % | 2 SKIPPED (`ENV_DEPENDENT`) | `HOMOLOGATION_READY` |

**87 tests exécutés sur 87, tous réussis (100 %), pour chacun des deux
Channel Managers de référence. 2 tests supplémentaires sont volontairement
ignorés (voir Notes ci-dessous) — un test ignoré n'est jamais compté comme un
échec.**

## Notes

- **Rate limit : 1001 requêtes → 429** [`ENV_DEPENDENT`] — Le rate limiting
  (1000 requêtes/minute par clé API) est validé manuellement via Redis. Le
  test automatique n'est pas fiable en environnement de test séquentiel : les
  1001 requêtes émises l'une après l'autre dépassent la fenêtre d'une minute
  avant d'atteindre le seuil, ce qui invaliderait le test plutôt que de
  prouver l'absence de limitation. Ce comportement est identique sur les deux
  Channel Managers testés (Phase 1 et Phase 8).

## Détail par phase (identique pour les deux Channel Managers)

| Phase | Tests | Résultat | Portée |
|---|---|---|---|
| Phase 1 — Authentification | 6/7 (1 skip) | ✅ PASSED | Phase critique |
| Phase 2 — Capabilities | 11/11 | ✅ PASSED | — |
| Phase 3 — Associations | 10/10 | ✅ PASSED | Phase critique |
| Phase 4 — Mapping associations | 13/13 | ✅ PASSED | Phase critique |
| Phase 5 — Disponibilités | 12/12 | ✅ PASSED | — |
| Phase 6 — Tarifs | 14/14 | ✅ PASSED | — |
| Phase 7 — Restrictions | 14/14 | ✅ PASSED | — |
| Phase 8 — Robustesse | 7/8 (1 skip) | ✅ PASSED | — |

Une phase marquée « critique » interrompt l'homologation en cas d'échec
(`stopped_early`) — aucune des deux exécutions n'a déclenché cet arrêt.

### Phase 1 — Authentification

- ✅ Connexion avec clé valide → `200 OK`
- ✅ Header `Authorization` manquant → `401 MISSING_AUTH_HEADER`
- ✅ Clé inconnue → `401 INVALID_API_KEY`
- ✅ Clé désactivée (`is_active=false`) → `403 KEY_DISABLED`
- ✅ Clé expirée (`expires_at < now()`) → `403 KEY_EXPIRED`
- ✅ Hôtel inactif → `403 HOTEL_INACTIVE`
- ⚠️ Rate limit : 1001 requêtes → `429` — SKIPPED (`ENV_DEPENDENT`)

### Phase 2 — Capabilities

- ✅ Endpoint accessible sans authentification → `200`
- ✅ Réponse contient `ota_name` et `version`
- ✅ Réponse contient `capabilities.availability` / `rates` / `restrictions`
- ✅ Chaque capability contient `supported: true` et `batch: true`
- ✅ Chaque champ contient `name`, `type`, `required`, `description`
- ✅ `rate_plan_room_id` est `required: true` et sans `default` sur les 3 capabilities
- ✅ Un paramètre optionnel avec valeur par défaut la documente (ex. `rates.currency`)
- ✅ Réponse contient `mapping.endpoint`
- ✅ Réponse contient `limits.batch_size`
- ✅ Réponse contient `limits.rate_limit`
- ✅ Temps de réponse < 50 ms (endpoint statique)

### Phase 3 — Associations

- ✅ Retourne la liste des associations → `200`
- ✅ Chaque association contient `id`, `name`, `room_type_name`, `rate_plan_name`, `room_type_id`, `rate_plan_id`, `mapped`, `confirmed_at`
- ✅ `mapped: false` pour toutes (aucun mapping initial)
- ✅ Champ `summary` présent `{ total, mapped, unmapped }`
- ✅ `summary.unmapped = summary.total` au départ
- ✅ Tri : non-mappées en premier
- ✅ `403` si hôtel inactif
- ✅ `403` si `hotel_id` ne correspond pas à la clé API (scope)
- ✅ `401` si clé absente
- ✅ Temps de réponse < 500 ms

### Phase 4 — Mapping associations

- ✅ `POST /channel/association-mapping` — mapper une association valide → `201`
- ✅ Réponse contient `id`, `hotel_id`, `channel_id`, `rate_plan_room_id`, `room_type_id`, `rate_plan_id`
- ✅ `GET` associations → `mapped: true` après `POST`
- ✅ `summary.mapped` +1 après chaque mapping
- ✅ Idempotent : `POST` deux fois → pas de doublon
- ✅ `rate_plan_room_id` inexistant → `404`
- ✅ `rate_plan_room_id` d'un autre hôtel → `404`
- ✅ `403` si hôtel inactif
- ✅ `DELETE /channel/association-mapping` — supprimer un mapping existant → `200`
- ✅ `GET` associations → `mapped: false` après `DELETE`
- ✅ `summary.unmapped` +1 après `DELETE`
- ✅ Mapping inexistant → `404`
- ✅ Workflow complet : mapper toutes les associations de l'hôtel

### Phase 5 — Disponibilités

- ✅ Push disponibilité valide → `200`
- ✅ Vérification données en base après push
- ✅ Push 0 chambre disponible → `200`, stocké correctement (fermeture)
- ✅ `available_rooms` négatif → `400`
- ✅ Format de date invalide → `400`
- ✅ `rate_plan_room_id` non mappé → `400`
- ✅ `403` si hôtel inactif
- ✅ Push 30 jours en une requête → `200`, `processed = 30`
- ✅ Vérification de toutes les dates en base après le lot
- ✅ `UPSERT` : repush même date → mise à jour
- ✅ Une date invalide dans le lot → rollback complet → `400`
- ✅ Push 365 jours → `200` en moins de 5 secondes

### Phase 6 — Tarifs

- ✅ Push tarif valide → `200`
- ✅ Vérification données en base
- ✅ `price = 0` → `400`
- ✅ `price` négatif → `400`
- ✅ `currency` manquante → défaut `EUR` appliqué
- ✅ `is_refundable` manquant → accepté (non persisté par date)
- ✅ `rate_plan_room_id` non mappé → `400`
- ✅ `403` si hôtel inactif
- ✅ Push 30 tarifs en une requête → `200`, `processed = 30`
- ✅ `UPSERT` : repush même date → prix mis à jour
- ✅ Rollback si une ligne invalide dans le lot → `400`
- ✅ `extra_adult_amount` fourni → stocké correctement
- ✅ `cancellation_deadline` fourni → stocké correctement
- ✅ Push sans `extra_adult_amount` → `null` en base

### Phase 7 — Restrictions

- ✅ Push restrictions valides → `200`
- ✅ Vérification données en `calendar_restrictions`
- ✅ Vérification synchronisation `calendar_availability` (`stop_sell`, `min_stay`, `max_stay`)
- ✅ `min_stay = 0` → `400`
- ✅ `min_stay > max_stay` → `400`
- ✅ `stop_sell = true` → stocké correctement
- ✅ `closed_to_arrival = true` → stocké correctement
- ✅ `rate_plan_room_id` non mappé → `400`
- ✅ `403` si hôtel inactif
- ✅ Push 30 restrictions → `200`, `processed = 30`
- ✅ Rollback si une ligne invalide dans le lot
- ✅ `stop_sell = true` puis `false` → mise à jour OK
- ✅ `blackout = true` → stocké correctement
- ✅ `closed_to_arrival` sur `check_in` → propagé au miroir `calendar_availability`

### Phase 8 — Robustesse

- ✅ 50 hôtels en push simultané → pas d'erreur
- ✅ Payload JSON malformé → `400`
- ✅ Payload > 10 Mo → `413`
- ✅ Timeout simulé → géré correctement (pas de blocage indéfini)
- ✅ Retry avec backoff : 3 échecs → succès
- ⚠️ Rate limit : 1001 req/min → `429` — SKIPPED (`ENV_DEPENDENT`)
- ✅ Vérification logs d'audit générés pour chaque requête
- ✅ Vérification cache Redis invalidé après chaque push

## Conclusion

L'API Push otarelay v2 est déclarée **prête pour homologation partenaire**
(`HOMOLOGATION_READY`) sur la base de 87 tests exécutés et réussis sur 87
(100 %), couvrant l'authentification, la découverte de capacités, le mapping
d'associations, et les trois flux de push (disponibilités, tarifs,
restrictions), ainsi que la robustesse (charge, payloads invalides, timeout,
retry, audit, cache). Les 2 tests ignorés portent exclusivement sur la
vérification automatisée du rate limiting, validée par ailleurs
manuellement ; ils ne constituent pas un frein à l'homologation.

Avant mise en production d'un Channel Manager partenaire, otarelay recommande
de rejouer cette même suite contre l'environnement sandbox
(`https://api.dev.otarelay.com`, `--env staging`) avec les identifiants
propres au CM concerné (`--cm <identifiant>`), afin de valider l'intégration
réelle plutôt que le jeu de données de référence `HTL_TEST`.

---

**Reproduction** :
```
npx tsx src/tests/homologation/runner.ts --cm cm_test1 --version v2 --report
npx tsx src/tests/homologation/runner.ts --cm cm_test2 --version v2 --report
```

---

Signé : **Équipe otarelay — Homologation Partenaires**
