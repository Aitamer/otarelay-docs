# Workflow d'intégration — API Push otarelay v2

L'intégration d'un Channel Manager (CM) avec otarelay suit trois phases
séquentielles. Chaque phase dépend du résultat de la précédente : la
Phase 2 nécessite les identifiants découverts en Phase 1, la Phase 3
nécessite les associations confirmées en Phase 2.

```
Phase 1 : Discovery ──► Phase 2 : Mapping ──► Phase 3 : Push quotidien
(une fois)              (par hôtel connecté)   (en continu)
```

Tous les exemples ci-dessous utilisent `fetch` (Node.js 18+ / navigateur) et
supposent une clé API déjà obtenue (voir [`sandbox.md`](./sandbox.md)).

---

## Phase 1 — Discovery

**Objectif** : découvrir le contrat de l'API avant d'écrire le moindre code
de mapping. Cette phase ne nécessite pas de clé API et n'est à réaliser
qu'une seule fois (ou à chaque montée de version de l'intégration).

### Appel

```js
const BASE_URL = 'https://api.dev.otarelay.com';

async function discoverCapabilities() {
  const res = await fetch(`${BASE_URL}/channel/capabilities`);
  if (!res.ok) {
    throw new Error(`Discovery failed: ${res.status}`);
  }
  return res.json();
}

const capabilities = await discoverCapabilities();
console.log(capabilities.limits.batch_size); // 366
console.log(capabilities.capabilities.rates.fields); // liste des champs attendus par /channel/rates
```

### Ce qu'il faut en retenir

- `capabilities.availability|rates|restrictions.fields` décrit précisément
  chaque champ attendu (type, requis, défaut, contraintes min/max) — à
  utiliser pour générer ou valider votre mapping de champs internes.
- `capabilities.limits.batch_size` et `capabilities.limits.rate_limit`
  dimensionnent vos appels par lot et votre stratégie de retry.
- Aucune authentification n'est nécessaire à cette étape.

---

## Phase 2 — Mapping

**Objectif** : pour chaque hôtel connecté, déterminer quelles associations
chambre × plan tarifaire (`rate_plan_room_id`) votre CM doit gérer, puis les
confirmer une par une. Cette phase se répète à chaque nouvel hôtel connecté,
ou lorsque l'hôtelier modifie sa grille chambres/plans tarifaires côté
otarelay.

### 2.1 — Lister les associations disponibles

```js
async function listAssociations(hotelId, apiKey) {
  const res = await fetch(
    `${BASE_URL}/channel/room-type-rate-plan-associations?hotel_id=${encodeURIComponent(hotelId)}`,
    { headers: { Authorization: `Bearer ${apiKey}` } },
  );
  if (!res.ok) {
    throw new Error(`List associations failed: ${res.status}`);
  }
  return res.json();
}

const { associations, summary } = await listAssociations('HTL_TEST', apiKey);
console.log(`${summary.unmapped}/${summary.total} associations à mapper`);
```

### 2.2 — Confirmer (mapper) chaque association gérée

```js
async function confirmMapping(hotelId, ratePlanRoomId, apiKey) {
  const res = await fetch(`${BASE_URL}/channel/association-mapping`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${apiKey}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({ hotel_id: hotelId, rate_plan_room_id: ratePlanRoomId }),
  });
  if (!res.ok) {
    throw new Error(`Mapping failed for ${ratePlanRoomId}: ${res.status}`);
  }
  return res.json(); // 201 — { id, hotel_id, channel_id, rate_plan_room_id, room_type_id, rate_plan_id, confirmed_at }
}

// Mapper toutes les associations non mappées d'un coup
const unmapped = associations.filter((a) => !a.mapped);
for (const association of unmapped) {
  await confirmMapping('HTL_TEST', association.id, apiKey);
}
```

### 2.3 — Retirer un mapping (association non reprise par votre CM)

```js
async function removeMapping(hotelId, ratePlanRoomId, apiKey) {
  const res = await fetch(`${BASE_URL}/channel/association-mapping`, {
    method: 'DELETE',
    headers: { Authorization: `Bearer ${apiKey}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({ hotel_id: hotelId, rate_plan_room_id: ratePlanRoomId }),
  });
  if (!res.ok) {
    throw new Error(`Unmapping failed: ${res.status}`);
  }
  return res.json(); // 200 — { message, rate_plan_room_id, mapped: false }
}
```

### Ce qu'il faut en retenir

- `POST /channel/association-mapping` est **idempotent** : ré-appeler sur une
  association déjà mappée ne crée pas de doublon, elle est simplement
  reconfirmée (`confirmed_at` mis à jour).
- Seules les associations mappées acceptent des pushes en Phase 3 — un push
  sur une association non mappée est rejeté en `400 ASSOCIATION_NOT_MAPPED`.

---

## Phase 3 — Push quotidien

**Objectif** : synchroniser en continu disponibilités, tarifs et restrictions
pour les associations mappées en Phase 2. Les trois endpoints sont
indépendants (à appeler séparément) et acceptent chacun un objet unique ou un
tableau (push par lot, jusqu'à `capabilities.limits.batch_size` entrées).

### 3.1 — Disponibilités

```js
async function pushAvailability(entries, apiKey) {
  const res = await fetch(`${BASE_URL}/channel/availability`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${apiKey}`, 'Content-Type': 'application/json' },
    body: JSON.stringify(entries), // objet unique ou tableau — tous avec le même hotel_id
  });
  if (!res.ok) {
    throw new Error(`Availability push failed: ${res.status}`);
  }
  return res.json(); // { status: 'ok', processed: N, failed: 0 }
}

await pushAvailability(
  [
    { hotel_id: 'HTL_TEST', rate_plan_room_id: 42, date: '2026-09-01', available_rooms: 5 },
    { hotel_id: 'HTL_TEST', rate_plan_room_id: 42, date: '2026-09-02', available_rooms: 3 },
  ],
  apiKey,
);
```

### 3.2 — Tarifs

```js
async function pushRates(entries, apiKey) {
  const res = await fetch(`${BASE_URL}/channel/rates`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${apiKey}`, 'Content-Type': 'application/json' },
    body: JSON.stringify(entries),
  });
  if (!res.ok) {
    throw new Error(`Rates push failed: ${res.status}`);
  }
  return res.json();
}

await pushRates(
  { hotel_id: 'HTL_TEST', rate_plan_room_id: 42, date: '2026-09-01', price: 129.9, currency: 'EUR' },
  apiKey,
);
```

### 3.3 — Restrictions

```js
async function pushRestrictions(entries, apiKey) {
  const res = await fetch(`${BASE_URL}/channel/restrictions`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${apiKey}`, 'Content-Type': 'application/json' },
    body: JSON.stringify(entries),
  });
  if (!res.ok) {
    throw new Error(`Restrictions push failed: ${res.status}`);
  }
  return res.json();
}

await pushRestrictions(
  { hotel_id: 'HTL_TEST', rate_plan_room_id: 42, date: '2026-12-24', min_stay: 2, closed_to_arrival: true },
  apiKey,
);
```

### Bonnes pratiques de push

- **Tout-ou-rien par lot** : une entrée invalide dans un tableau annule
  l'intégralité du lot (aucune écriture partielle). Validez vos payloads côté
  CM avant l'envoi pour éviter des lots systématiquement rejetés.
- **Idempotence par date** : chaque push est un `UPSERT` — repousser la même
  date met simplement à jour la valeur existante, sans erreur ni doublon.
- **Respect du rate limit** : 1000 requêtes/minute par clé API
  (`capabilities.limits.rate_limit`). Un dépassement retourne `429` avec un
  en-tête `Retry-After` (secondes avant réessai).
- **Retry avec backoff exponentiel** recommandé sur toute erreur `429` ou
  `5xx` transitoire.

### Exemple de boucle de synchronisation complète

```js
async function dailySync(hotelId, apiKey, availabilityEntries, rateEntries, restrictionEntries) {
  await pushAvailability(availabilityEntries, apiKey);
  await pushRates(rateEntries, apiKey);
  await pushRestrictions(restrictionEntries, apiKey);
}
```

---

Pour la liste exhaustive des champs et des codes d'erreur, voir
[`api-reference.md`](./api-reference.md) et [`errors.md`](./errors.md).
