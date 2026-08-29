# FAQ technique — otarelay Channel API v2

## 1. Pourquoi mes pushes échouent-ils systématiquement en `400 ASSOCIATION_NOT_MAPPED` ?

Le `rate_plan_room_id` utilisé n'a pas été confirmé pour votre canal. Les
endpoints `/channel/availability`, `/channel/rates` et
`/channel/restrictions` n'acceptent que des associations préalablement
mappées via `POST /channel/association-mapping` (Phase 2 du
[workflow](./workflow.md)). Vérifiez le statut `mapped` via
`GET /channel/room-type-rate-plan-associations` avant de pousser.

## 2. Puis-je envoyer un objet unique au lieu d'un tableau sur les endpoints de push ?

Oui. `POST /channel/availability`, `/channel/rates` et `/channel/restrictions`
acceptent indifféremment un objet JSON unique ou un tableau d'objets. Le
comportement (validation, transaction, réponse) est identique — un tableau
d'un seul élément est équivalent à un objet unique.

## 3. Quelle est la taille maximale d'un lot ?

`capabilities.limits.batch_size` (366 dans la réponse de
`GET /channel/capabilities`) documente la taille de référence, alignée sur le
plus grand lot validé en homologation (une année complète, 365 jours). Aucune
limite dure n'est imposée par la validation du corps de requête au-delà de la
taille globale de la requête (voir `413`), mais restez raisonnable pour
limiter la durée de traitement et le risque de rollback complet sur une seule
ligne invalide.

## 4. Que se passe-t-il si une seule date est invalide dans un lot de 200 ?

Le lot entier est rejeté (`400`), aucune des 200 entrées n'est écrite. Les
endpoints de push sont transactionnels « tout ou rien » — validez vos données
côté CM avant l'envoi, ou fractionnez vos lots par petits groupes pour limiter
l'impact d'une ligne fautive.

## 5. Puis-je repousser deux fois la même date sans erreur ?

Oui. Chaque entrée de disponibilité/tarif/restriction est un `UPSERT` sur la
clé `(hotel_id, room_type, rate_plan, date)` — repousser écrase simplement la
valeur précédente, sans erreur ni doublon. C'est le mécanisme normal de
synchronisation quotidienne.

## 6. Pourquoi `is_refundable` que j'envoie sur `/channel/rates` n'apparaît-il jamais dans mes vérifications ?

`is_refundable` est accepté dans le payload pour compatibilité avec certains
CM, mais **n'est pas persisté par date** : c'est un attribut statique du plan
tarifaire dans le référentiel otarelay (configuré côté hôtelier), pas une
valeur variable par jour. Le champ est simplement ignoré s'il est fourni.

## 7. Comment savoir si ma clé API a expiré ou a été désactivée ?

Les deux cas retournent `403`, mais avec des codes distincts :
`KEY_EXPIRED` (date d'expiration dépassée) ou `KEY_DISABLED` (désactivée
manuellement côté otarelay). Voir [`errors.md`](./errors.md#403--forbidden).
Dans les deux cas, contactez votre point de contact otarelay pour une
régénération.

## 8. J'obtiens `403` alors que ma clé API est valide et active — pourquoi ?

Vérifiez que le `hotel_id` envoyé dans le corps de la requête correspond
exactement à celui associé à votre clé API. Une clé est scopée sur un couple
unique `(hotel_id, channel_id)` : toute requête portant un `hotel_id`
différent est rejetée, même avec une clé par ailleurs valide.

## 9. Comment gérer la limite de 1000 requêtes/minute proprement ?

Surveillez l'en-tête `Retry-After` renvoyé sur une réponse `429` et
respectez-le avant de réessayer. Pour éviter d'atteindre la limite,
privilégiez le push par lot (un seul appel pour plusieurs dates) plutôt que
des appels unitaires répétés, et lissez vos synchronisations dans le temps
plutôt que de les émettre en rafale.

## 10. Le endpoint `GET /channel/capabilities` nécessite-t-il une clé API ?

Non. C'est le seul endpoint public de l'intégration — accessible sans
authentification, pour permettre la découverte automatique du contrat de
l'API avant même d'avoir obtenu une clé. Tous les autres endpoints
`/channel/*` requièrent l'en-tête `Authorization: Bearer <clé>`.

---

Question non couverte ici ? Consultez [`errors.md`](./errors.md) pour le
détail des codes HTTP, ou contactez l'équipe otarelay via votre point de
contact d'intégration.
