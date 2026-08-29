# Homologation otarelay — Guide de démarrage rapide

Ce dossier regroupe la documentation destinée à un Channel Manager (CM) qui
intègre l'API Push v2 d'**otarelay** : référence des endpoints, workflow
d'intégration, gestion des erreurs, environnement sandbox, FAQ, rapport
d'homologation officiel, et une collection de tests prête à l'emploi
(Bruno et Postman) pour valider concrètement votre intégration.

## Intégration otarelay en 4 étapes

```
┌─────────────────┐     ┌──────────────────┐     ┌───────────────────┐     ┌──────────────────┐
│ 1. Découverte    │ →   │ 2. Mapping        │ →   │ 3. Push quotidien  │ →   │ 4. Homologation    │
│                  │     │                   │     │                    │     │                    │
│ GET /channel/    │     │ GET associations  │     │ POST availability  │     │ npx tsx runner.ts  │
│ capabilities     │     │ POST association- │     │ POST rates         │     │ --cm <id> --report │
│ (sans clé API)   │     │ mapping           │     │ POST restrictions  │     │                    │
└─────────────────┘     └──────────────────┘     └───────────────────┘     └──────────────────┘
```

### Étape 1 — Découverte (`GET /channel/capabilities`)

Endpoint public, sans authentification. Il expose le contrat de l'API :
champs attendus, types, contraintes, valeurs par défaut, limites
(`batch_size`, `rate_limit`). C'est le point d'entrée pour générer un client
ou valider votre mapping de champs avant toute intégration.

### Étape 2 — Mapping (`GET`/`POST /channel/association-mapping`)

Chaque hôtel expose des associations chambre × plan tarifaire
(`rate_plan_room_id`). Un CM récupère la liste via
`GET /channel/room-type-rate-plan-associations`, puis confirme (« mappe »)
chaque association qu'il gère via `POST /channel/association-mapping`. Seules
les associations mappées acceptent des pushes.

### Étape 3 — Push quotidien (disponibilités, tarifs, restrictions)

Trois endpoints indépendants, tous acceptant un objet unique ou un tableau
(push par lot) : `POST /channel/availability`, `POST /channel/rates`,
`POST /channel/restrictions`. Chaque appel est transactionnel — un enregistrement
invalide dans un lot annule tout le lot (aucune écriture partielle).

### Étape 4 — Homologation

Avant la mise en production, faites valider votre intégration par la suite de
tests automatisée (89 tests, 8 phases). Voir
[`homologation-report.md`](./homologation-report.md) pour le résultat de
référence, [`sandbox.md`](./sandbox.md) pour l'environnement de test, et
[`tests/`](./tests/) pour rejouer les mêmes scénarios côté client (Bruno ou
Postman) avant de nous transmettre votre rapport.

## Sommaire du dossier

| Fichier | Contenu |
|---|---|
| [`api-reference.md`](./api-reference.md) | Référence complète des 7 endpoints (méthode, auth, paramètres, réponses, erreurs) |
| [`workflow.md`](./workflow.md) | Détail des 3 phases d'intégration avec exemples de code |
| [`errors.md`](./errors.md) | Tous les codes HTTP retournés par l'API, cause et solution |
| [`sandbox.md`](./sandbox.md) | Environnement de test, obtention d'une clé API, données de test |
| [`faq.md`](./faq.md) | Questions techniques fréquentes |
| [`homologation-report.md`](./homologation-report.md) | Rapport officiel des résultats de la suite d'homologation v2 |
| [`tests/bruno/`](./tests/bruno/) | Collection Bruno complète (20 requêtes, 7 phases) — voir [`tests/bruno/README-tests.md`](./tests/bruno/README-tests.md) |
| [`tests/postman/`](./tests/postman/) | Collection Postman équivalente (`otarelay-collection.json`, schema v2.1) |

## Tester votre intégration (Bruno / Postman)

Deux collections prêtes à l'emploi, avec les mêmes 20 requêtes et les mêmes
assertions automatiques, réparties sur les 7 phases de l'homologation
(authentification, capabilities, associations, mapping, disponibilités,
tarifs, restrictions) :

- **Bruno** (recommandé, léger et open-source) :
  [`tests/bruno/`](./tests/bruno/) — voir
  [`tests/bruno/README-tests.md`](./tests/bruno/README-tests.md) pour
  l'installation, la configuration de l'environnement sandbox et le
  lancement.
- **Postman** : [`tests/postman/otarelay-collection.json`](./tests/postman/otarelay-collection.json)
  — collection v2.1, à importer directement (*Import* → sélectionner le
  fichier), variables de collection identiques à l'environnement Bruno
  (`base_url`, `api_key`, `hotel_id`).

Dans les deux cas : renseignez votre clé API sandbox et votre `hotel_id`,
lancez les phases dans l'ordre, vérifiez que tous les tests passent, puis
exportez et transmettez le rapport à otarelay pour validation finale.

## Convention de nommage

- **CM** : Channel Manager (le système partenaire qui pousse les données vers otarelay)
- **`rate_plan_room_id`** : identifiant pivot d'une association chambre × plan
  tarifaire — clé de voûte de tous les endpoints de push
- **`channel_id`** : identifiant du canal, résolu automatiquement depuis votre
  clé API (jamais fourni dans un payload)

## Support

Pour toute question technique non couverte par ce dossier, consultez d'abord
[`faq.md`](./faq.md), puis contactez l'équipe otarelay via votre point de
contact d'intégration habituel.
