# Collection Bruno — Homologation otarelay Channel API v2

Cette collection reproduit, en boîte noire (HTTP pur, sans accès à la base de
données), les mêmes scénarios que la suite d'homologation interne
(`src/tests/homologation/`) documentée dans
[`../../homologation-report.md`](../../homologation-report.md). Elle sert de
support pour qu'un Channel Manager valide sa propre intégration avant la mise
en production.

Une version Postman équivalente est disponible dans
[`../postman/otarelay-collection.json`](../postman/otarelay-collection.json).

## Prérequis

- Installer Bruno : [https://usebruno.com](https://usebruno.com)
- Obtenir une **API Key sandbox** depuis votre contact otarelay (voir
  [`../../sandbox.md`](../../sandbox.md) — les clés ne sont pas
  auto-générées, elles sont émises par un administrateur otarelay pour un
  couple hôtel/canal donné)

## Configuration

1. Ouvrir ce dossier (`docs/homologation/tests/bruno/`) comme collection dans
   Bruno (*Open Collection*).
2. Sélectionner l'environnement **`otarelay-sandbox`** en haut à droite.
3. Éditer l'environnement et renseigner :

   | Variable | Description |
   |---|---|
   | `base_url` | Déjà pré-rempli : `https://api.dev.otarelay.com` |
   | `api_key` | Votre clé API sandbox (obtenue auprès d'otarelay) |
   | `hotel_id` | L'identifiant de l'hôtel de test associé à votre clé |
   | `api_key_disabled` | *Optionnel* — uniquement pour `phase1-auth/04-disabled-key.bru` (voir ci-dessous) |
   | `api_key_expired` | *Optionnel* — uniquement pour `phase1-auth/05-expired-key.bru` (voir ci-dessous) |

   > ⚠️ **Ne committez jamais une vraie clé API dans ce fichier.** Le
   > placeholder livré dans `environments/otarelay-sandbox.bru` est un texte
   > d'exemple à écraser localement, jamais à pousser en dépôt une fois
   > remplacé par une valeur réelle. Si vous versionnez vos environnements
   > Bruno, dupliquez ce fichier en une copie non suivie par Git plutôt que de
   > modifier l'original.

4. `api_key_disabled` / `api_key_expired` : ces deux tests vérifient un
   comportement que seul un administrateur otarelay peut préparer côté
   backoffice (désactiver une clé, ou émettre une clé déjà expirée). Demandez
   ces deux clés à votre contact otarelay uniquement si vous voulez exécuter
   `phase1-auth/04` et `05` — le reste de la collection ne les utilise pas.

## Structure de la collection

```
phase1-auth/              5 requêtes — authentification (200/401/403)
phase2-capabilities/      1 requête  — découverte du contrat API (public)
phase3-associations/      2 requêtes — lecture du référentiel chambre × plan
phase4-mapping/           3 requêtes — cycle mapper → vérifier → démapper
phase5-availability/      3 requêtes — push disponibilités (unitaire, lot, erreur)
phase6-rates/             3 requêtes — push tarifs (unitaire, lot, erreur)
phase7-restrictions/      3 requêtes — push restrictions (unitaire, valeur, erreur)
```

Chaque requête `.bru` contient : méthode + URL, en-têtes, corps JSON le cas
échéant, et des assertions automatiques (bloc `assert` pour le code HTTP et
les champs simples, bloc `tests` pour les vérifications de structure/valeurs
via `expect`).

### Enchaînement automatique entre les requêtes

La collection est conçue pour être **rejouable de bout en bout sans étape
manuelle** :

- `phase3-associations/01-get-associations-empty.bru` capture dynamiquement
  deux identifiants dans des variables d'exécution (`rate_plan_room_id` et
  `rate_plan_room_id_push`) à partir de la réponse réelle de votre hôtel de
  test — aucun `rate_plan_room_id` n'est codé en dur.
- `phase4-mapping/` utilise `rate_plan_room_id` pour démontrer le cycle
  complet mapper → vérifier → **démapper** (le test 03 termine
  volontairement l'association en état non-mappé, pour vérifier `DELETE`).
- `phase5/6/7` (push) utilisent une association **différente**
  (`rate_plan_room_id_push`) et reconfirment son mapping automatiquement via
  un script *pre-request* sur leur premier test (`01-*.bru`) — idempotent, sans
  effet de bord — afin de ne jamais dépendre de l'état laissé par la phase 4.
- Les dates poussées sont calculées dynamiquement (aujourd'hui + N jours,
  plage distincte par test) plutôt que codées en dur, pour que la collection
  reste valide indéfiniment sans édition manuelle.

## Lancement

1. **Lancer chaque phase dans l'ordre** (1 → 7), soit requête par requête,
   soit dossier par dossier via *Run Collection* / *Run Folder* dans Bruno.
2. **Vérifier que tous les tests passent** (onglet *Tests*/*Assertions* de
   chaque requête, ou le résumé du *Collection Runner*).
3. **Exporter le rapport Bruno** : dans le *Collection Runner*, après
   exécution, utilisez *Export Results* (JSON) pour générer un rapport
   horodaté.
4. **Envoyer le rapport à otarelay** via votre point de contact d'intégration,
   accompagné du `channel_id` utilisé, pour validation finale avant mise en
   production.

## En cas d'échec

Consultez [`../../errors.md`](../../errors.md) pour la cause et la solution
associées à chaque code HTTP retourné par une requête en échec.
