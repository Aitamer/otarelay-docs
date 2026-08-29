# Guide sandbox — otarelay Channel API v2

## URL

```
https://api.dev.otarelay.com
```

Environnement de test dédié à l'intégration et à l'homologation, isolé des
données de production. Tous les endpoints documentés dans
[`api-reference.md`](./api-reference.md) y sont disponibles à l'identique.

## Comment obtenir une clé API

Les clés API du flux Push ne sont **pas auto-générées** par le CM : elles
sont émises côté otarelay, pour un couple (hôtel, canal) donné, par un
administrateur via le backoffice ou l'endpoint interne
`POST /api/v1/hotels/:hotelId/api-keys`.

1. Contactez votre point de contact otarelay pour l'ouverture d'un hôtel de
   test sur le sandbox (ou utilisez `HTL_TEST`, voir ci-dessous).
2. otarelay génère la clé et vous la communique **une seule fois** — la clé
   en clair n'est jamais stockée côté serveur (seul son hash SHA-256 est
   conservé) et ne peut donc pas être récupérée a posteriori. Conservez-la
   immédiatement dans votre gestionnaire de secrets.
3. Si la clé est perdue ou compromise, demandez une régénération plutôt
   qu'une nouvelle création : la régénération conserve le même `channel_id`
   et invalide l'ancien secret.

Chaque clé est scopée sur un couple unique (`hotel_id`, `channel_id`) : toute
requête `/channel/*` portant un `hotel_id` différent de celui associé à la
clé est rejetée en `403`.

## Données de test disponibles

L'hôtel de référence utilisé par la suite d'homologation est `HTL_TEST`,
préconfiguré avec :

| Référentiel | Contenu |
|---|---|
| Chambres (`room_type`) | `DBL` (Double), `TWN` (Twin), `TPL` (Triple) |
| Plans tarifaires (`rate_plan`) | `BAR` (Best Available Rate), `NRF` (Non-Refundable), `BB` (Bed & Breakfast) |
| Associations pivot (`rate_plan_room`) | 9 combinaisons chambre × plan (3 × 3), toutes non mappées initialement pour un nouveau canal |

Cet hôtel et ce jeu de données sont ceux exercés par
`npx tsx src/tests/homologation/runner.ts` (voir
[`homologation-report.md`](./homologation-report.md)) — vous pouvez vous en
inspirer pour structurer vos propres scénarios de test, ou demander à
otarelay un hôtel de test dédié à votre canal (`--cm <votre_identifiant>`).

## Comment réinitialiser le sandbox

Le sandbox ne dispose pas de réinitialisation globale automatique — chaque
hôtel de test est indépendant. Deux options :

- **Réinitialiser un hôtel de test spécifique** : les mappings, disponibilités,
  tarifs et restrictions sont propres à chaque association pivot ; repousser
  une valeur écrase simplement la précédente (upsert), sans besoin de purge
  préalable. Pour repartir d'un état totalement vierge, demandez à otarelay
  la suppression de l'hôtel de test (script interne `delete:hotel`, cascade
  sur toutes les données liées) puis sa recréation.
- **Retirer un mapping ponctuel** : `DELETE /channel/association-mapping`
  (voir [`api-reference.md`](./api-reference.md)) pour repartir d'une
  association non mappée sans toucher au reste du référentiel.

## Durée de validité

Par convention, les clés API émises sur le sandbox ont une **validité de
30 jours** à compter de leur création ou dernière régénération. Passé ce
délai, toute requête authentifiée avec cette clé est rejetée en `403
KEY_EXPIRED` (voir [`errors.md`](./errors.md)). Demandez une régénération à
l'approche de l'échéance pour éviter toute interruption de vos tests
d'intégration.

---

Une fois votre intégration validée sur ce sandbox, passez à l'étape
d'homologation formelle décrite dans
[`homologation-report.md`](./homologation-report.md).
