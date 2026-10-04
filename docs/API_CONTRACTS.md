# Contrats d'API

> **Version 0 — brouillon.** À valider en Phase 0, quand on aura l'export du logiciel de Yossef et la décision sur les prix. Rien n'est figé avant la porte G0.
>
> **4 octobre 2026 :** ajout des contrats de la Mini App chauffeur (J à P), de l'inscription Telegram (Q) et de l'événement `ride_status_changed` (R) ; D, E, F et G ajustés (décisions D-011 à D-016). **À relire par Ilan avant fusion.**

Chaque module peut changer de technologie sans casser les autres, tant que ces contrats restent stables.

**Mocks :** chaque exemple JSON de ce fichier est une réponse valide. Ilan peut développer le bot et la Mini App contre ces exemples ; Oren vérifie que ses réponses ont exactement la même forme.

## Règles

1. Le propriétaire d'un contrat est le module qui le **sert** (presque toujours `backend/`).
2. Toute modification passe par une pull request qui modifie ce fichier, relue par le propriétaire **et** par le module qui consomme.
3. Un changement cassant incrémente la version (`x-contract-version`) et est annoncé avant fusion.
4. Les réponses sont déterministes : pas de prix ni de statut inventés côté appelant.
5. L'identité de l'appelant vient de son authentification, jamais du corps de la requête.

## Conventions communes

| Sujet | Règle |
| --- | --- |
| Transport | HTTPS, JSON, UTF-8 |
| Authentification entre modules | En-tête `x-driverss-key` : un secret par module appelant (`voice`, `dispatch`), stocké en variable d'environnement. Côté serveur uniquement : jamais dans le dashboard ni dans la Mini App |
| Authentification du dashboard | Session Supabase (clé `anon` + RLS) ; les actions critiques passent par les endpoints ci-dessous |
| Authentification des chauffeurs (Mini App) | `Authorization: Bearer <access_token>`, jeton obtenu par J. Fonctions préfixées `driver-` |
| Identité de l'acteur | Déduite de l'authentification : session du sadran, jeton du chauffeur, ou `telegram_user_id` reçu par le webhook vérifié du bot |
| Version | En-tête `x-contract-version: 0` |
| Argent | Entiers en agorot : `price_agorot: 18000` = 180 ₪ |
| Dates | ISO 8601 avec décalage : `2026-10-04T08:00:00+03:00` |
| Téléphones | E.164 : `+9725XXXXXXXX` |
| Idempotence | Les créations acceptent `idempotency_key` : le même appel répété ne crée qu'une seule course. Le claim est idempotent pour un même chauffeur (`replay`) |
| Succès | `{ "ok": true, ... }` |
| Erreur | `{ "ok": false, "error": { "code": "unknown_place", "message": "..." } }`. `message` sert au débogage : les interfaces affichent leurs propres textes selon `code` |
| Codes HTTP | `200` succès ; `400` requête invalide ; `401` authentification absente ou invalide ; `403` chauffeur non lié ou bloqué ; `404` introuvable ; `409` conflit d'état (`already_taken`, `not_claimable`) ; `429` trop de requêtes |

## Vue d'ensemble

| # | Contrat | Appelant | Servi par | Phase |
| --- | --- | --- | --- | --- |
| A | `POST /lookup-customer` | voice | backend | 2 |
| B | `POST /get-quote` | voice, dashboard | backend | 1 |
| C | `POST /create-ride` | dashboard, voice | backend | 1 |
| D | Événement `ride_ready_for_dispatch` | backend | dispatch (bot) | 1 |
| E | `POST /claim-ride` | dispatch (bot) | backend | 1 |
| F | `POST /assign-ride` | dispatch (dashboard) | backend | 1 |
| G | `POST /update-ride-status` | dispatch (dashboard) | backend | 1 |
| H | `GET /rides/{ride_id}` | voice, dispatch | backend | 1 |
| I | Webhook de fin d'appel | ElevenLabs | backend | 2 |
| J | `POST /driver-auth-telegram` | dispatch (Mini App) | backend | 1 |
| K | `GET /driver-me` | dispatch (Mini App) | backend | 1 |
| L | `POST /driver-set-availability` | dispatch (Mini App) | backend | 1 |
| M | `GET /driver-rides` | dispatch (Mini App) | backend | 1 |
| N | `GET /driver-rides/{ride_id}` | dispatch (Mini App) | backend | 1 |
| O | `POST /driver-claim-ride` | dispatch (Mini App) | backend | 1 |
| P | Canaux temps réel chauffeur | dispatch (Mini App), abonnée | backend (Realtime) | 1 |
| Q | `POST /link-driver-telegram` | dispatch (bot) | backend | 1 |
| R | Événement `ride_status_changed` | backend | dispatch (bot) | 1 |

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
  "event_id": "uuid",
  "ride_id": 153,
  "station_id": "st_driverss",
  "from_city": "בני ברק", "from_area": "מרכז",
  "to_city": "ירושלים", "to_area": "רמות",
  "pickup_time": "2026-10-04T08:00:00+03:00",
  "kind": "passengers",
  "passengers": 2,
  "price_agorot": 18000,
  "message_text": "...message au format actuel des chauffeurs, en hébreu...",
  "claim_window_seconds": 60,
  "recipients": [
    { "driver_id": "uuid", "telegram_user_id": 123456789 }
  ]
}
```

Réponse attendue du dispatch : `{ "ok": true }` (course acceptée pour diffusion).

- `recipients` : les chauffeurs à notifier, calculés par le backend (voir « Éligibilité » dans [`ARCHITECTURE.md`](ARCHITECTURE.md)). Le bot envoie un message privé à chacun, sans ajouter ni retirer personne. Relance à 60 s : aux mêmes destinataires ; ensuite, le sadran attribue.
- `event_id` : identifiant de la ligne `ride_events` à l'origine de l'événement. Livraison au moins une fois : le bot ignore un `event_id` déjà traité.

## E. `POST /claim-ride`

Un chauffeur clique « je prends » dans le bot. Opération atomique : un seul gagnant. Appelant : `dispatch/bot`, avec `x-driverss-key`.

```json
{ "ride_id": 153, "telegram_user_id": 123456789 }
```

`telegram_user_id` est l'auteur du clic, tel que reçu par le webhook Telegram vérifié. Le backend retrouve le chauffeur ; le bot n'envoie jamais de `driver_id`.

Succès : `{ "ok": true, "replay": false, "ride": { ... } }`, où `ride` est un objet `driver_ride` avec `driver_view: "mine"` et ses `pickup_details` — exemple complet en O.

```json
{ "ok": false, "error": { "code": "already_taken", "message": "ride 153 is already claimed" } }
```

Codes : `already_taken`, `not_claimable` (annulée, terminée, pas encore diffusée), `driver_blocked`, `driver_not_linked` (ce compte Telegram n'est lié à aucun chauffeur de la station), `not_found`.

Le bot utilise la réponse pour répondre au clic (« הנסיעה שלך! » ou « הנסיעה כבר נלקחה »). Le message privé avec les détails part à la réception de R, quel que soit le canal du claim : une seule voie, pas de doublon.

> Changements par rapport à la première v0 : `driver_id` remplacé par `telegram_user_id` (le bot n'avait aucun moyen de connaître `driver_id`) ; erreur au format commun au lieu de `reason` ; réponse identique à O.

## F. `POST /assign-ride`

Le sadran attribue à la main depuis le dashboard. Même fonction atomique que E.

```json
{ "ride_id": 153, "driver_id": "uuid" }
```

Le sadran est identifié par sa session : `assigned_by` n'est plus dans le corps. Réponses : comme O ; le chauffeur n'a pas besoin d'être lié à Telegram. Le chauffeur est prévenu par R (message privé du bot) et par le signal `ride_assigned`.

## G. `POST /update-ride-status`

Clore une course.

```json
{ "ride_id": 153, "status": "done", "reason": null }
```

`status` vaut `done`, `cancelled` ou `no_show`. L'acteur est déduit de la session : `actor_id` n'est plus dans le corps. Transitions autorisées : voir [`DATABASE.md`](DATABASE.md). Le crédit de 7 % et la commission sont écrits au grand livre uniquement au passage à `done`. L'annulation d'une course déjà prise prévient le chauffeur (R, signal `ride_cancelled`).

## H. `GET /rides/{ride_id}`

Lire l'état d'une course (pour informer le client ou le sadran).

```json
{ "ok": true, "ride_id": 153, "status": "claimed", "driver": { "id": "uuid", "display_name": "..." } }
```

## I. Webhook de fin d'appel (ElevenLabs → backend)

Phase 2. Signature vérifiée. Le backend stocke : `conversation_id`, durée, transcription, résultat (`ride_created`, `transferred`, `failed`), raison d'échec, lien vers l'enregistrement privé.

---

## Mini App chauffeur — conventions (J à P)

- Appelant : `dispatch/miniapp`, sur le téléphone du chauffeur. Aucun secret : seulement l'URL du projet, la clé `anon` publique et le jeton de session.
- Toutes les fonctions sauf J exigent `Authorization: Bearer <access_token>`. Le chauffeur vient du jeton ; son statut est relu à chaque appel.
- Erreurs communes à K, L, M, N et O : `unauthorized` (401, jeton absent ou expiré), `driver_blocked` (403).
- CORS : seule l'origine de la Mini App est acceptée.
- Les textes affichés sont choisis par la Mini App selon `code`, en hébreu ([`MINIAPP.md`](MINIAPP.md)).

### Objet `driver_ride`

La vue d'une course pour un chauffeur. C'est le backend qui décide de ce qui y figure.

```json
{
  "ride_id": 153,
  "driver_view": "available",
  "from_city": "בני ברק", "from_area": "מרכז",
  "to_city": "ירושלים", "to_area": "רמות",
  "pickup_time": "2026-10-04T08:00:00+03:00",
  "kind": "passengers",
  "passengers": 2,
  "price_agorot": 18000,
  "message_text": "...message au format actuel des chauffeurs, en hébreu...",
  "posted_at": "2026-10-04T07:41:12+03:00",
  "updated_at": "2026-10-04T07:41:12+03:00",
  "pickup_details": null
}
```

| `driver_view` | Sens | Champs présents |
| --- | --- | --- |
| `available` | Course `posted` que ce chauffeur peut prendre | Tous, avec `pickup_details: null` |
| `mine` | Course `claimed` par ce chauffeur | Tous, avec `pickup_details` |
| `taken` | Prise par un autre chauffeur | `ride_id` et `driver_view` seulement |
| `closed` | Annulée, terminée ou client absent | `ride_id` et `driver_view` seulement |

`pickup_details`, seulement pour `mine` :

```json
{ "address_text": "...", "customer_phone": "+9725XXXXXXXX", "notes": "..." }
```

## J. `POST /driver-auth-telegram`

Échanger l'initData de Telegram contre une session. Appelée une fois à chaque ouverture de la Mini App, sans jeton (seulement la clé `anon` du projet).

```json
{ "init_data": "query_id=...&user=...&auth_date=...&hash=..." }
```

`init_data` est la chaîne brute `Telegram.WebApp.initData`, envoyée telle quelle, dans le corps — jamais dans l'URL.

```json
{
  "ok": true,
  "session": {
    "access_token": "eyJ...",
    "refresh_token": "...",
    "expires_at": "2026-10-04T08:42:00+03:00"
  },
  "driver": {
    "driver_id": "uuid",
    "display_name": "...",
    "phone_e164": "+9725XXXXXXXX",
    "status": "active",
    "availability": { "available": true, "until": null }
  },
  "station": {
    "station_id": "st_driverss",
    "brand_name": "Driverss",
    "driver_support_phone_e164": "+9727XXXXXXXX"
  }
}
```

La Mini App passe `access_token` et `refresh_token` à supabase-js (`setSession`, avec `persistSession: false`) ; supabase-js rafraîchit le jeton et le transmet à Realtime. `driver` et `station` : comme K, pour éviter un aller-retour au démarrage.

| Code | HTTP | Quand |
| --- | --- | --- |
| `invalid_init_data` | 401 | Signature fausse, champ manquant, autre bot |
| `init_data_expired` | 401 | `auth_date` trop vieux (plus de 300 s) |
| `driver_not_linked` | 403 | Aucun chauffeur de la station lié à ce compte Telegram |
| `driver_blocked` | 403 | Chauffeur bloqué |

Idempotence : chaque appel crée une nouvelle session ; l'appeler deux fois est sans risque. Seul effet en base : la création de l'utilisateur Supabase du chauffeur, une seule fois, à sa première connexion.

## K. `GET /driver-me`

Profil du chauffeur connecté.

```json
{
  "ok": true,
  "driver": {
    "driver_id": "uuid",
    "display_name": "...",
    "phone_e164": "+9725XXXXXXXX",
    "status": "active",
    "availability": { "available": true, "until": null }
  },
  "station": {
    "station_id": "st_driverss",
    "brand_name": "Driverss",
    "driver_support_phone_e164": "+9727XXXXXXXX"
  },
  "server_time": "2026-10-04T07:42:00+03:00"
}
```

- `availability.available` est la disponibilité effective : une fin dépassée donne `false`.
- `brand_name` vient de la station : la Mini App n'écrit jamais « Driverss » en dur (marque neutre, Phase 4).
- `server_time` permet d'afficher « dans 25 min » juste, même si l'horloge du téléphone ne l'est pas.

## L. `POST /driver-set-availability`

```json
{ "available": true, "until": "2026-10-04T23:00:00+03:00" }
```

```json
{ "available": false }
```

```json
{ "ok": true, "availability": { "available": true, "until": "2026-10-04T23:00:00+03:00" } }
```

- `until` est optionnel. S'il est présent : dans le futur et au plus 24 h plus tard, sinon `invalid_until` (400).
- `available: false` efface `until`.
- Effet : seulement sur les destinataires des prochaines notifications (D). Ne retire ni une course déjà prise, ni un message déjà envoyé.
- Idempotent : le même appel deux fois donne le même état. Deux appareils en même temps : la dernière écriture gagne.

## M. `GET /driver-rides`

`?scope=available` : courses `posted` de la station du chauffeur, triées par `pickup_time` croissant, 50 au plus.
`?scope=mine` : courses `claimed` par ce chauffeur, triées par `pickup_time` croissant, avec `pickup_details`.

```json
{
  "ok": true,
  "server_time": "2026-10-04T07:42:00+03:00",
  "rides": [
    {
      "ride_id": 153,
      "driver_view": "available",
      "from_city": "בני ברק", "from_area": "מרכז",
      "to_city": "ירושלים", "to_area": "רמות",
      "pickup_time": "2026-10-04T08:00:00+03:00",
      "kind": "passengers",
      "passengers": 2,
      "price_agorot": 18000,
      "message_text": "...",
      "posted_at": "2026-10-04T07:41:12+03:00",
      "updated_at": "2026-10-04T07:41:12+03:00",
      "pickup_details": null
    }
  ]
}
```

Erreurs : `invalid_scope` (400), plus les erreurs communes.

Concurrence : la liste est une photo, qui peut être périmée une seconde plus tard. C'est le claim qui tranche, jamais la liste.

## N. `GET /driver-rides/{ride_id}`

Une course, par exemple ouverte depuis le bouton « פתח » du bot.

```json
{ "ok": true, "server_time": "2026-10-04T07:43:05+03:00", "ride": { "ride_id": 153, "driver_view": "taken" } }
```

Une course d'une autre station ou inexistante → `not_found` (404), sans dire laquelle des deux.

## O. `POST /driver-claim-ride`

Le chauffeur appuie sur « אני לוקח » dans la Mini App. Même fonction atomique que E et F.

```json
{ "ride_id": 153 }
```

```json
{
  "ok": true,
  "replay": false,
  "ride": {
    "ride_id": 153,
    "driver_view": "mine",
    "from_city": "בני ברק", "from_area": "מרכז",
    "to_city": "ירושלים", "to_area": "רמות",
    "pickup_time": "2026-10-04T08:00:00+03:00",
    "kind": "passengers",
    "passengers": 2,
    "price_agorot": 18000,
    "message_text": "...",
    "posted_at": "2026-10-04T07:41:12+03:00",
    "updated_at": "2026-10-04T07:42:03+03:00",
    "pickup_details": { "address_text": "...", "customer_phone": "+9725XXXXXXXX", "notes": "..." }
  }
}
```

| Code | HTTP | Quand |
| --- | --- | --- |
| `already_taken` | 409 | Un autre chauffeur l'a prise avant |
| `not_claimable` | 409 | Course annulée, close, ou pas encore diffusée |
| `not_found` | 404 | Course inexistante ou d'une autre station |
| `driver_blocked` | 403 | Chauffeur bloqué |

- Idempotence : le même chauffeur qui rappelle pour la même course reçoit `ok: true` avec `replay: true`, sans nouvel événement. La Mini App peut donc réessayer après une erreur réseau.
- Jamais de nouvel essai automatique après `already_taken` ou `not_claimable`.
- Après un succès : signaux P, et événement R vers le bot.

## P. Canaux temps réel chauffeur

Servis par le backend (Supabase Realtime, Broadcast). La Mini App s'abonne avec supabase-js, en canal privé (`config: { private: true }`), avec son jeton de session.

| Canal | Qui peut écouter | Événement | Charge utile |
| --- | --- | --- | --- |
| `rides:{station_id}` | Chauffeurs `active` de la station | `ride_available` | `{ "ride_id": 153, "at": "2026-10-04T07:41:12+03:00" }` |
| `rides:{station_id}` | Idem | `ride_unavailable` | `{ "ride_id": 153, "reason": "claimed", "at": "2026-10-04T07:42:03+03:00" }` — `reason` : `claimed` ou `cancelled` |
| `driver:{driver_id}` | Ce chauffeur | `ride_assigned` | `{ "ride_id": 153, "via": "dashboard", "at": "2026-10-04T07:42:03+03:00" }` — `via` : `telegram_bot`, `miniapp` ou `dashboard` |
| `driver:{driver_id}` | Ce chauffeur | `ride_cancelled` | `{ "ride_id": 153, "at": "2026-10-04T07:50:00+03:00" }` |
| `driver:{driver_id}` | Ce chauffeur | `ride_closed` | `{ "ride_id": 153, "status": "done", "at": "2026-10-04T09:10:00+03:00" }` — `status` : `done` ou `no_show` |

- Les signaux ne portent jamais de donnée client et ne sont pas l'état : la Mini App relit M ou N.
- Aucune garantie d'ordre ni de livraison : relire à l'abonnement, au retour au premier plan, et toutes les 20 s tant que le canal n'est pas abonné.
- La Mini App n'émet rien sur ces canaux.

---

## Q. `POST /link-driver-telegram`

Lier un compte Telegram à un chauffeur pré-inscrit. Appelant : `dispatch/bot`, avec `x-driverss-key`, quand le chauffeur partage son contact.

```json
{ "station_id": "st_driverss", "telegram_user_id": 123456789, "phone_e164": "+9725XXXXXXXX" }
```

Avant d'appeler, le bot vérifie que le contact partagé est bien celui de l'expéditeur (`contact.user_id` égal à `from.id`) et normalise le numéro en E.164.

```json
{ "ok": true, "driver_id": "uuid", "display_name": "...", "status": "active", "already_linked": false }
```

| Code | Quand |
| --- | --- |
| `driver_not_found` | Aucun chauffeur pré-inscrit avec ce numéro dans la station : le bot dit d'appeler le sadran |
| `telegram_already_linked` | Ce chauffeur est lié à un autre compte Telegram, ou ce compte à un autre chauffeur : le sadran doit délier |
| `driver_blocked` | Chauffeur bloqué |

Idempotence : la même paire deux fois → `ok: true` avec `already_linked: true`.

## R. Événement `ride_status_changed`

Envoyé par le backend au bot à chaque changement de statut après `posted`, quel que soit le canal d'origine.

```json
{
  "event": "ride_status_changed",
  "event_id": "uuid",
  "ride_id": 153,
  "station_id": "st_driverss",
  "status": "claimed",
  "previous_status": "posted",
  "via": "miniapp",
  "driver": { "driver_id": "uuid", "telegram_user_id": 123456789 },
  "pickup_details": { "address_text": "...", "customer_phone": "+9725XXXXXXXX", "notes": "..." },
  "at": "2026-10-04T07:42:03+03:00"
}
```

- `status` : `claimed`, `cancelled`, `done` ou `no_show`. `via` seulement pour `claimed`.
- `driver` : nul si aucun chauffeur ; `telegram_user_id` nul si le chauffeur n'est pas lié.
- `pickup_details` : seulement pour `claimed`, et seulement pour le message privé du gagnant.
- Ce que fait le bot : `claimed` → message privé au gagnant avec les détails, « נלקחה » sur les messages des autres destinataires ; `cancelled` → si un chauffeur était attribué, message privé « הנסיעה בוטלה », et mise à jour des messages encore ouverts ; `done`, `no_show` → rien d'obligatoire.
- Livraison au moins une fois : le bot ignore un `event_id` déjà traité. Ordre non garanti : en cas de doute, le bot relit H.

Réponse attendue : `{ "ok": true }`.

---

## Ce qui reste interne à un module

- Le webhook Telegram (Telegram → `dispatch/bot`) est interne au dispatch. Il vérifie l'en-tête secret envoyé par Telegram, puis appelle E ou Q.
- Les boutons des messages du bot, les écrans de la Mini App et la correspondance « course → messages envoyés » (pour les modifier après un claim) sont internes à `dispatch/`.
- Le prompt et la configuration de l'agent vocal sont internes à `voice/`.
