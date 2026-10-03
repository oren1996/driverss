# Contrats d'API

> **Version 0 — brouillon.** À valider en Phase 0, quand on aura l'export du logiciel de Yossef et la décision sur les prix. Rien n'est figé avant la porte G0.

Chaque module peut changer de technologie sans casser les autres, tant que ces contrats restent stables.

## Règles

1. Le propriétaire d'un contrat est le module qui le **sert** (presque toujours `backend/`).
2. Toute modification passe par une pull request qui modifie ce fichier, relue par le propriétaire **et** par le module qui consomme.
3. Un changement cassant incrémente la version (`x-contract-version`) et est annoncé avant fusion.
4. Les réponses sont déterministes : pas de prix ni de statut inventés côté appelant.

## Conventions communes

| Sujet | Règle |
| --- | --- |
| Transport | HTTPS, JSON, UTF-8 |
| Authentification entre modules | En-tête `x-driverss-key` : un secret par module appelant (`voice`, `dispatch`), stocké en variable d'environnement |
| Authentification du dashboard | Session Supabase (clé `anon` + RLS) ; les actions critiques passent par les endpoints ci-dessous |
| Version | En-tête `x-contract-version: 0` |
| Argent | Entiers en agorot : `price_agorot: 18000` = 180 ₪ |
| Dates | ISO 8601 avec décalage : `2026-10-04T08:00:00+03:00` |
| Téléphones | E.164 : `+9725XXXXXXXX` |
| Idempotence | Les créations acceptent `idempotency_key` : le même appel répété ne crée qu'une seule course |
| Succès | `{ "ok": true, ... }` |
| Erreur | `{ "ok": false, "error": { "code": "unknown_place", "message": "..." } }` |

## Vue d'ensemble

| # | Contrat | Appelant | Servi par | Phase |
| --- | --- | --- | --- | --- |
| A | `POST /lookup-customer` | voice | backend | 2 |
| B | `POST /get-quote` | voice, dashboard | backend | 1 |
| C | `POST /create-ride` | dashboard, voice | backend | 1 |
| D | Événement `ride_ready_for_dispatch` | backend | dispatch | 1 |
| E | `POST /claim-ride` | dispatch (bot) | backend | 1 |
| F | `POST /assign-ride` | dispatch (dashboard) | backend | 1 |
| G | `POST /update-ride-status` | dispatch (dashboard) | backend | 1 |
| H | `GET /rides/{ride_id}` | voice, dispatch | backend | 1 |
| I | Webhook de fin d'appel | ElevenLabs | backend | 2 |

---

## A. `POST /lookup-customer`

Identifier le client à partir du numéro appelant.

```json
{ "station_id": "st_driverss", "caller_phone": "+9725XXXXXXXX" }
```

```json
{ "ok": true, "known": true, "customer_id": "uuid", "name": "...", "institution_code": "1234", "consent_needed": false }
```

## B. `POST /get-quote`

Calculer un prix. Ne crée rien.

```json
{
  "station_id": "st_driverss",
  "from_city": "בני ברק", "from_area": "מרכז",
  "to_city": "ירושלים", "to_area": "רמות",
  "pickup_time": "2026-10-04T08:00:00+03:00",
  "kind": "passengers",
  "passengers": 2
}
```

```json
{ "ok": true, "price_agorot": 18000, "credit_agorot": 1260, "from_place_id": "uuid", "to_place_id": "uuid" }
```

Erreurs : `unknown_place` (lieu non reconnu, avec suggestions), `no_price` (pas de prix pour ce trajet), `closed` (Shabbat ou fête).

## C. `POST /create-ride`

Créer une course. Le prix est recalculé et verrouillé par le backend : l'appelant n'envoie jamais de prix.

```json
{
  "idempotency_key": "dash-2026-10-04-0001",
  "station_id": "st_driverss",
  "source": "dashboard",
  "caller_phone": "+9725XXXXXXXX",
  "institution_code": "1234",
  "from_city": "בני ברק", "from_area": "מרכז",
  "to_city": "ירושלים", "to_area": "רמות",
  "pickup_address_text": "...",
  "pickup_time": "2026-10-04T08:00:00+03:00",
  "kind": "passengers",
  "passengers": 2,
  "notes": "",
  "conversation_id": null
}
```

```json
{ "ok": true, "ride_id": 153, "status": "created", "price_agorot": 18000 }
```

`source` vaut `dashboard` ou `voice`. `conversation_id` est rempli par la voix pour la traçabilité.

## D. Événement `ride_ready_for_dispatch`

Envoyé par le backend au module dispatch quand une course passe à `posted`. **Jamais de numéro ni de nom de client dedans.**

```json
{
  "event": "ride_ready_for_dispatch",
  "ride_id": 153,
  "station_id": "st_driverss",
  "from_city": "בני ברק", "from_area": "מרכז",
  "to_city": "ירושלים", "to_area": "רמות",
  "pickup_time": "2026-10-04T08:00:00+03:00",
  "kind": "passengers",
  "passengers": 2,
  "price_agorot": 18000,
  "message_text": "...message au format actuel des chauffeurs, en hébreu...",
  "claim_window_seconds": 60
}
```

Réponse attendue du dispatch : `{ "ok": true }` (course acceptée pour diffusion).

## E. `POST /claim-ride`

Un chauffeur clique « je prends ». Opération atomique : un seul gagnant.

```json
{ "ride_id": 153, "driver_id": "uuid", "channel": "telegram" }
```

```json
{ "ok": true, "status": "claimed", "pickup_details": { "address_text": "...", "customer_phone": "+9725XXXXXXXX" } }
```

```json
{ "ok": false, "reason": "already_taken" }
```

Autres raisons : `not_claimable` (annulée, terminée…), `driver_blocked`. Les détails de prise en charge ne sont renvoyés qu'au gagnant, pour son message privé.

## F. `POST /assign-ride`

Le sadran attribue à la main depuis le dashboard. Même opération atomique que E.

```json
{ "ride_id": 153, "driver_id": "uuid", "assigned_by": "uuid-sadran" }
```

Réponses identiques à E.

## G. `POST /update-ride-status`

Clore une course.

```json
{ "ride_id": 153, "status": "done", "actor_id": "uuid", "reason": null }
```

`status` vaut `done`, `cancelled` ou `no_show`. Transitions autorisées : voir [`DATABASE.md`](DATABASE.md). Le crédit de 7 % et la commission sont écrits au grand livre uniquement au passage à `done`.

## H. `GET /rides/{ride_id}`

Lire l'état d'une course (pour informer le client ou le sadran).

```json
{ "ok": true, "ride_id": 153, "status": "claimed", "driver": { "id": "uuid", "display_name": "..." } }
```

## I. Webhook de fin d'appel (ElevenLabs → backend)

Phase 2. Signature vérifiée. Le backend stocke : `conversation_id`, durée, transcription, résultat (`ride_created`, `transferred`, `failed`), raison d'échec, lien vers l'enregistrement privé.

---

## Ce qui reste interne à un module

- Le webhook Telegram (Telegram → `dispatch/bot`) est interne au dispatch. Il vérifie l'en-tête secret envoyé par Telegram, puis appelle E.
- Le prompt et la configuration de l'agent vocal sont internes à `voice/`.
