# Référence API — otarelay Channel API v2

Base URL sandbox : `https://api.dev.otarelay.com` (voir [`sandbox.md`](./sandbox.md)).

Toutes les requêtes et réponses sont en JSON (`Content-Type: application/json`).
Sauf mention contraire, chaque endpoint requiert l'en-tête :

```
Authorization: Bearer <votre_clé_api>
```

Les endpoints de push (`/channel/availability`, `/channel/rates`,
`/channel/restrictions`) acceptent indifféremment un objet unique ou un
tableau d'objets dans le corps de la requête (push par lot). Tous les
éléments d'un même lot doivent porter le même `hotel_id`.

---

## GET /channel/capabilities

**Description** : Décrit le contrat de l'API — champs attendus par chaque
endpoint de push, types, contraintes, valeurs par défaut, et limites en
vigueur. Permet à un CM (ou un outil : portail partenaire, générateur de SDK,
documentation automatique) de découvrir automatiquement les paramètres
supportés avant de configurer son mapping et ses synchronisations.

**Authentification** : aucune (endpoint public).

**Paramètres** : aucun.

**Réponse `200 OK`** (extrait) :

```json
{
  "ota_name": "otarelay",
  "version": "2.0",
  "capabilities": {
    "availability": {
      "supported": true,
      "endpoint": "POST /channel/availability",
      "batch": true,
      "fields": [
        {
          "name": "rate_plan_room_id",
          "type": "integer",
          "required": true,
          "min": 1,
          "description": "Identifiant de l'association chambre × plan tarifaire, obtenu via GET /channel/room-type-rate-plan-associations et confirmé via POST /channel/association-mapping."
        },
        { "name": "date", "type": "string", "required": true, "description": "Date au format YYYY-MM-DD." },
        { "name": "available_rooms", "type": "integer", "required": true, "min": 0, "description": "Nombre de chambres disponibles pour cette date (0 = fermeture)." }
      ]
    },
    "rates": { "supported": true, "endpoint": "POST /channel/rates", "batch": true, "fields": ["..."] },
    "restrictions": { "supported": true, "endpoint": "POST /channel/restrictions", "batch": true, "fields": ["..."] }
  },
  "mapping": {
    "endpoint": "POST /channel/association-mapping",
    "delete_endpoint": "DELETE /channel/association-mapping",
    "list_endpoint": "GET /channel/room-type-rate-plan-associations"
  },
  "limits": {
    "batch_size": 366,
    "rate_limit": 1000
  }
}
```

**Codes d'erreur** : aucun cas d'erreur applicatif — l'endpoint est statique
et répond toujours `200`.

---

## GET /channel/room-type-rate-plan-associations

**Description** : Retourne la liste des associations chambre × plan
tarifaire (`rate_plan_room_id`) configurées pour un hôtel, avec leur statut
de mapping (`mapped: true/false`) pour le canal (`channel_id`) résolu depuis
votre clé API. Les associations non mappées apparaissent en premier.

**Authentification** : requise (`Authorization: Bearer <clé>`), la clé doit
être scopée sur le `hotel_id` demandé.

**Paramètres (query)** :

| Paramètre | Type | Requis | Description |
|---|---|---|---|
| `hotel_id` | string | oui | Identifiant otarelay de l'hôtel |

**Réponse `200 OK`** :

```json
{
  "hotel_id": "HTL_TEST",
  "channel_id": "ID_REF_Siteminder",
  "associations": [
    {
      "id": 42,
      "name": "Chambre Double - Best Available Rate DBL/BAR",
      "room_type_name": "Chambre Double",
      "rate_plan_name": "Best Available Rate",
      "room_type_id": "DBL",
      "rate_plan_id": "BAR",
      "mapped": false,
      "confirmed_at": null
    }
  ],
  "summary": { "total": 9, "mapped": 0, "unmapped": 9 }
}
```

**Codes d'erreur** :

| Code | Cas |
|---|---|
| `401` | Clé API absente, malformée ou inconnue |
| `403` | Hôtel inactif, ou clé API non scopée sur ce `hotel_id` |

---

## POST /channel/association-mapping

**Description** : Confirme (« mappe ») qu'un CM gère effectivement une
association chambre × plan tarifaire — précondition obligatoire avant de
pouvoir pousser de la disponibilité, des tarifs ou des restrictions pour
cette association. Opération **idempotente** : appeler deux fois avec le
même `rate_plan_room_id` ne crée pas de doublon (re-confirmation, `confirmed_at`
mis à jour).

**Authentification** : requise, clé scopée sur le `hotel_id` envoyé.

**Paramètres (body)** :

| Champ | Type | Requis | Description |
|---|---|---|---|
| `hotel_id` | string | oui | Identifiant otarelay de l'hôtel |
| `rate_plan_room_id` | integer | oui | Identifiant de l'association à mapper (obtenu via `GET /channel/room-type-rate-plan-associations`) |

**Requête** :

```json
{ "hotel_id": "HTL_TEST", "rate_plan_room_id": 42 }
```

**Réponse `201 Created`** :

```json
{
  "id": 17,
  "hotel_id": "HTL_TEST",
  "channel_id": "ID_REF_Siteminder",
  "rate_plan_room_id": 42,
  "room_type_id": "DBL",
  "rate_plan_id": "BAR",
  "confirmed_at": "2026-08-25T09:12:34.000Z"
}
```

**Codes d'erreur** :

| Code | Cas |
|---|---|
| `400` | `rate_plan_room_id` absent ou non numérique (validation du corps) |
| `401` | Clé API absente, malformée ou inconnue |
| `403` | Hôtel inactif, ou clé API non scopée sur ce `hotel_id` |
| `404` | `rate_plan_room_id` inexistant, ou appartenant à un autre hôtel |

---

## DELETE /channel/association-mapping

**Description** : Retire la confirmation de mapping d'une association pour
votre canal. Les pushes ultérieurs sur cette association seront rejetés
(`400 ASSOCIATION_NOT_MAPPED`) jusqu'à un nouveau `POST`.

**Authentification** : requise, clé scopée sur le `hotel_id` envoyé.

**Paramètres (body)** : identiques à `POST /channel/association-mapping`.

```json
{ "hotel_id": "HTL_TEST", "rate_plan_room_id": 42 }
```

**Réponse `200 OK`** :

```json
{ "message": "Mapping supprimé", "rate_plan_room_id": 42, "mapped": false }
```

**Codes d'erreur** :

| Code | Cas |
|---|---|
| `400` | `rate_plan_room_id` absent ou non numérique |
| `401` | Clé API absente, malformée ou inconnue |
| `403` | Hôtel inactif, ou clé API non scopée sur ce `hotel_id` |
| `404` | Aucun mapping actif pour ce `rate_plan_room_id` et ce canal |

---

## POST /channel/availability

**Description** : Pousse la disponibilité (nombre de chambres restantes) pour
une ou plusieurs dates. Accepte un objet unique ou un tableau (jusqu'à
365 jours en une requête, validé en homologation). Chaque entrée doit
référencer un `rate_plan_room_id` préalablement mappé (voir
`POST /channel/association-mapping`).

**Authentification** : requise, clé scopée sur le `hotel_id` de chaque entrée.

**Paramètres (body)** — objet ou tableau d'objets :

| Champ | Type | Requis | Défaut | Description |
|---|---|---|---|---|
| `hotel_id` | string | oui | — | Identifiant otarelay de l'hôtel (identique pour toutes les entrées d'un lot) |
| `rate_plan_room_id` | integer (≥ 1) | oui | — | Association chambre × plan tarifaire, mappée |
| `date` | string | oui | — | Date au format `YYYY-MM-DD` |
| `available_rooms` | integer (≥ 0) | oui | — | Chambres disponibles (`0` = fermeture) |

**Requête (lot)** :

```json
[
  { "hotel_id": "HTL_TEST", "rate_plan_room_id": 42, "date": "2026-09-01", "available_rooms": 5 },
  { "hotel_id": "HTL_TEST", "rate_plan_room_id": 42, "date": "2026-09-02", "available_rooms": 3 }
]
```

**Réponse `200 OK`** :

```json
{ "status": "ok", "processed": 2, "failed": 0 }
```

**Codes d'erreur** :

| Code | Cas |
|---|---|
| `400` | `available_rooms` négatif, `date` mal formatée, `rate_plan_room_id` non mappé, JSON malformé, ou une date invalide dans le lot (rollback complet) |
| `401` | Clé API absente, malformée ou inconnue |
| `403` | Hôtel inactif, ou clé API non scopée sur le `hotel_id` du lot |
| `413` | Corps de requête trop volumineux |
| `429` | Limite de débit dépassée (1000 req/min) |

---

## POST /channel/rates

**Description** : Pousse les tarifs pour une ou plusieurs dates, avec
options (devise, supplément adulte, échéance d'annulation). Même
fonctionnement par lot que `/channel/availability`.

**Authentification** : requise, clé scopée sur le `hotel_id` de chaque entrée.

**Paramètres (body)** — objet ou tableau d'objets :

| Champ | Type | Requis | Défaut | Description |
|---|---|---|---|---|
| `hotel_id` | string | oui | — | Identifiant otarelay de l'hôtel |
| `rate_plan_room_id` | integer (≥ 1) | oui | — | Association chambre × plan tarifaire, mappée |
| `date` | string | oui | — | Date au format `YYYY-MM-DD` |
| `price` | decimal (> 0) | oui | — | Prix par nuit, strictement positif |
| `currency` | string (3 lettres) | non | `"EUR"` | Code devise ISO 4217 |
| `extra_adult_amount` | decimal (≥ 0) | non | `null` | Supplément par adulte additionnel |
| `cancellation_deadline` | datetime ISO 8601 | non | `null` | Date limite d'annulation gratuite |
| `is_refundable` | boolean | non | — | Accepté pour compatibilité CM mais **non persisté** : `is_refundable` reste un attribut statique du plan tarifaire côté référentiel otarelay, ignoré silencieusement si fourni ici |

**Requête** :

```json
{
  "hotel_id": "HTL_TEST",
  "rate_plan_room_id": 42,
  "date": "2026-09-01",
  "price": 129.90,
  "currency": "EUR",
  "extra_adult_amount": 15,
  "cancellation_deadline": "2026-08-30T23:59:59.000Z"
}
```

**Réponse `200 OK`** :

```json
{ "status": "ok", "processed": 1, "failed": 0 }
```

**Codes d'erreur** :

| Code | Cas |
|---|---|
| `400` | `price` nul ou négatif, `date` mal formatée, `rate_plan_room_id` non mappé, JSON malformé, ou une ligne invalide dans le lot (rollback complet) |
| `401` | Clé API absente, malformée ou inconnue |
| `403` | Hôtel inactif, ou clé API non scopée sur le `hotel_id` du lot |
| `413` | Corps de requête trop volumineux |
| `429` | Limite de débit dépassée (1000 req/min) |

---

## POST /channel/restrictions

**Description** : Pousse les restrictions de vente (durée de séjour,
fermeture de vente, blocages arrivée/départ, black-out) pour une ou
plusieurs dates. Les champs `stop_sell`, `closed_to_arrival` et
`closed_to_departure` sont également répercutés dans le calendrier de
disponibilité (miroir interne, transparent pour le CM).

**Authentification** : requise, clé scopée sur le `hotel_id` de chaque entrée.

**Paramètres (body)** — objet ou tableau d'objets :

| Champ | Type | Requis | Défaut | Description |
|---|---|---|---|---|
| `hotel_id` | string | oui | — | Identifiant otarelay de l'hôtel |
| `rate_plan_room_id` | integer (≥ 1) | oui | — | Association chambre × plan tarifaire, mappée |
| `date` | string | oui | — | Date au format `YYYY-MM-DD` |
| `min_stay` | integer (≥ 1) | non | — (aucune contrainte) | Durée de séjour minimale, en nuits |
| `max_stay` | integer (≥ 1) | non | — (aucune contrainte) | Durée de séjour maximale, en nuits (doit être ≥ `min_stay`) |
| `stop_sell` | boolean | non | `false` | Ferme la vente pour cette date |
| `closed_to_arrival` | boolean | non | `false` | Interdit l'arrivée (check-in) à cette date |
| `closed_to_departure` | boolean | non | `false` | Interdit le départ (check-out) à cette date |
| `blackout` | boolean | non | `false` | Marque la date comme black-out |

**Requête** :

```json
{
  "hotel_id": "HTL_TEST",
  "rate_plan_room_id": 42,
  "date": "2026-12-24",
  "min_stay": 2,
  "stop_sell": false,
  "closed_to_arrival": true
}
```

**Réponse `200 OK`** :

```json
{ "status": "ok", "processed": 1, "failed": 0 }
```

**Codes d'erreur** :

| Code | Cas |
|---|---|
| `400` | `min_stay = 0`, `min_stay > max_stay`, `date` mal formatée, `rate_plan_room_id` non mappé, JSON malformé, ou une ligne invalide dans le lot (rollback complet) |
| `401` | Clé API absente, malformée ou inconnue |
| `403` | Hôtel inactif, ou clé API non scopée sur le `hotel_id` du lot |
| `413` | Corps de requête trop volumineux |
| `429` | Limite de débit dépassée (1000 req/min) |

---

Pour le détail générique de chaque code HTTP (cause, message, solution), voir
[`errors.md`](./errors.md).
