# Gestion des erreurs — API Push otarelay v2

## Format des réponses d'erreur

La majorité des endpoints `/channel/*` retournent une enveloppe applicative
standard :

```json
{ "error": "CODE_ERREUR", "message": "Description lisible", "details": { "...": "contexte additionnel" } }
```

Deux exceptions à connaître :

- **Échecs d'authentification** (`401`/`403` levés par le middleware de clé
  API, avant même d'atteindre le handler de route) utilisent une enveloppe
  légèrement différente :
  ```json
  { "error": { "code": "INVALID_API_KEY", "message": "API key not found", "http_status": 401 } }
  ```
- **Erreurs de validation Zod** (`400`) et **JSON malformé** (`400` levé par
  le framework HTTP avant même le parsing applicatif) suivent le format
  générique de leur couche respective — voir le détail par code ci-dessous.

## Tableau des codes HTTP

| Code | Cause | Message retourné | Solution recommandée |
|---|---|---|---|
| **200** | Requête traitée avec succès (`GET`, `DELETE`, ou push `POST`) | `{ "status": "ok", "processed": N, "failed": 0 }` (push) ou corps spécifique à l'endpoint | Aucune action — traiter la réponse normalement |
| **201** | Ressource créée (`POST /channel/association-mapping`) | Objet du mapping créé (`id`, `hotel_id`, `channel_id`, `rate_plan_room_id`, ...) | Aucune action |
| **400** | Corps de requête invalide : champ manquant/mal typé (validation Zod), JSON malformé, `available_rooms`/`price` négatif ou nul, `min_stay > max_stay`, date au mauvais format, ou `rate_plan_room_id` non mappé (`ASSOCIATION_NOT_MAPPED`) | `{ "error": "VALIDATION_ERROR", "message": "Invalid request input", "details": {...} }` (validation) ou `{ "error": "ASSOCIATION_NOT_MAPPED", "message": "Association non mappée", "details": {...} }` (mapping manquant) | Corriger le payload selon `details`, ou confirmer d'abord le mapping via `POST /channel/association-mapping` avant de pousser des données |
| **401** | En-tête `Authorization` absent ou mal formé (`MISSING_AUTH_HEADER`), ou clé API inconnue en base (`INVALID_API_KEY`) | `{ "error": { "code": "MISSING_AUTH_HEADER" \| "INVALID_API_KEY", "message": "...", "http_status": 401 } }` | Vérifier la présence et le format `Authorization: Bearer <clé>` ; vérifier que la clé n'a pas été régénérée côté otarelay |
| **403** | Clé API désactivée (`KEY_DISABLED`), expirée (`KEY_EXPIRED`), hôtel inactif (`HOTEL_INACTIVE`), ou clé non scopée sur le `hotel_id` demandé (`INVALID_API_KEY`) | `{ "error": { "code": "KEY_DISABLED" \| "KEY_EXPIRED" \| "HOTEL_INACTIVE", ... } }` ou `{ "error": "INVALID_API_KEY", "message": "API key is not scoped to this hotel" }` | Contacter l'équipe otarelay pour réactiver/renouveler la clé, ou vérifier que le `hotel_id` envoyé correspond bien à celui associé à la clé |
| **404** | `rate_plan_room_id` inexistant ou appartenant à un autre hôtel (`RATE_PLAN_ROOM_NOT_FOUND`), ou mapping inexistant lors d'un `DELETE` (`CHANNEL_ASSOCIATION_MAPPING_NOT_FOUND`) | `{ "error": "RATE_PLAN_ROOM_NOT_FOUND" \| "CHANNEL_ASSOCIATION_MAPPING_NOT_FOUND", "message": "...", "details": {...} }` | Vérifier l'identifiant via `GET /channel/room-type-rate-plan-associations` avant de mapper ou de démapper |
| **409** | Conflit d'unicité — n'est pas déclenché par les 7 endpoints homologués (idempotents par upsert) ; s'applique aux endpoints legacy de mapping par code (`/channel/room-type-mapping`, `/channel/rate-plan-mapping`, dépréciés) et à la création de clé API (`DUPLICATE_API_KEY`) | `{ "error": "DUPLICATE_MAPPING" \| "DUPLICATE_API_KEY", "message": "...", "details": {...} }` | Ne pas recréer une ressource déjà existante ; utiliser l'endpoint de régénération le cas échéant |
| **413** | Corps de requête supérieur à la limite acceptée par le serveur | Réponse générée par le framework HTTP (hors enveloppe applicative `{error,message}`) | Réduire la taille du lot envoyé (respecter `capabilities.limits.batch_size`) et fractionner les envois volumineux |
| **429** | Limite de débit dépassée : plus de 1000 requêtes/minute pour cette clé API (`RATE_LIMIT_EXCEEDED`) | `{ "error": "RATE_LIMIT_EXCEEDED", "message": "Rate limit of 1000 requests/min exceeded", "details": { "retryAfterSeconds": N, "limit": 1000 } }`, en-tête `Retry-After: <N>` | Respecter l'en-tête `Retry-After` avant de réessayer ; lisser les appels (batching, file d'attente) plutôt que d'émettre en rafale |

## Détail par code

### 200 — OK

Traitement réussi. Pour les endpoints de push, le corps confirme le nombre
d'entrées traitées (`processed`) ; `failed` reste toujours `0` sur un `200`
— tout échec partiel provoque un rollback complet du lot et une réponse
`400`, jamais un `200` avec `failed > 0`.

### 201 — Created

Réservé à `POST /channel/association-mapping`. Un second appel avec le même
`rate_plan_room_id` retourne également `201` (upsert idempotent), pas de
`409`.

### 400 — Bad Request

Cause la plus fréquente en intégration. Trois familles distinctes :

1. **Validation de schéma** (Zod) : champ requis absent, type incorrect,
   contrainte min/max violée (ex. `price` ≤ 0, `min_stay = 0`,
   `min_stay > max_stay`, format de date invalide). Le champ `details`
   précise le ou les champs en cause.
2. **JSON malformé** : corps de requête qui n'est pas un JSON valide. Rejeté
   par le framework HTTP avant même d'atteindre la validation applicative.
3. **Association non mappée** (`ASSOCIATION_NOT_MAPPED`) : le
   `rate_plan_room_id` envoyé n'a pas été confirmé via
   `POST /channel/association-mapping` pour votre canal. Sur un lot, un seul
   `rate_plan_room_id` non mappé fait échouer l'ensemble de la requête
   (rollback).

### 401 — Unauthorized

Absence totale d'identité exploitable : en-tête `Authorization` manquant ou
mal formé (`MISSING_AUTH_HEADER`), ou clé API syntaxiquement valide mais
absente de la base otarelay (`INVALID_API_KEY`). À distinguer de `403`, où
l'identité est reconnue mais l'accès est refusé.

### 403 — Forbidden

L'identité est reconnue mais l'accès est refusé :

- `KEY_DISABLED` — la clé a été désactivée côté backoffice otarelay.
- `KEY_EXPIRED` — la date d'expiration de la clé est dépassée.
- `HOTEL_INACTIVE` — l'hôtel associé à la clé est désactivé sur la
  plateforme.
- Scope invalide — le `hotel_id` transmis dans le corps ne correspond pas à
  celui associé à la clé API utilisée.

### 404 — Not Found

Référence inconnue dans le référentiel otarelay :

- `RATE_PLAN_ROOM_NOT_FOUND` — `rate_plan_room_id` inexistant, ou existant
  mais rattaché à un autre hôtel que celui demandé.
- `CHANNEL_ASSOCIATION_MAPPING_NOT_FOUND` — tentative de `DELETE` sur un
  mapping qui n'a jamais été confirmé (ou déjà supprimé).

### 409 — Conflict

Non déclenché par les 7 endpoints de l'intégration standard (tous conçus en
upsert idempotent). Réservé aux endpoints annexes : création de clé API sur
un `channel_id` déjà utilisé (`DUPLICATE_API_KEY`) et aux endpoints de
mapping legacy dépréciés.

### 413 — Payload Too Large

Le corps de la requête dépasse la taille maximale acceptée par le serveur.
Contrairement aux autres codes, la réponse n'est pas nécessairement au format
`{error, message, details}` applicatif : elle peut être générée directement
par le framework HTTP avant que la requête n'atteigne le code métier.
Fractionnez les envois volumineux en plusieurs lots plus petits.

### 429 — Too Many Requests

Plus de 1000 requêtes en une minute glissante pour la même clé API. La
réponse inclut un en-tête `Retry-After` (en secondes) — attendez ce délai
avant de réessayer plutôt que de renvoyer immédiatement.

---

Voir [`api-reference.md`](./api-reference.md) pour les erreurs spécifiques à
chaque endpoint et [`faq.md`](./faq.md) pour les cas d'usage fréquents.
