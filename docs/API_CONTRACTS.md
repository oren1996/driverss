# Contrats d'API

> **Version 0 — brouillon.** À valider en Phase 0, quand on aura l'export du logiciel de Yossef et la décision sur les prix. Rien n'est figé avant la porte G0.
>
> **4 octobre 2026 :** ajout des contrats de la Mini App chauffeur (J à P), de l'inscription Telegram (Q) et de l'événement `ride_status_changed` (R) ; D, E, F et G ajustés (décisions D-011 à D-016).
>
> **7 octobre 2026 (relecture) :** tableau « Qui peut appeler quoi » ; devis identifié (B, C) ; `version` des courses ; D et R passent par une file d'envois que le bot lit (T) ; gestion des chauffeurs (S) ; `expected_version` sur G. Propositions D-017 à D-021. **À relire par Ilan ; Eitan à prévenir pour C.**
>
> **7 octobre 2026 (deuxième relecture) :** envois en annonces et corrections, chaque message connu corrigé, confirmation `unknown` et envois « peut-être partis » (D, R, T) ; liaisons Telegram, déliaison et nouvelle liaison contrôlée par un admin (J à O, Q, S, D-022) ; `version` dans toutes les vues (N). **À relire par Ilan.**

Chaque module peut changer de technologie sans casser les autres, tant que ces contrats restent stables.

**Mocks :** chaque exemple JSON de ce fichier est une réponse valide. Ilan peut développer le bot, le dashboard et la Mini App contre ces exemples ; Oren vérifie que ses réponses ont exactement la même forme.

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
| Authentification entre modules | En-tête `x-driverss-key` : un secret par module appelant (`voice`, `dispatch`), stocké en variable d'environnement. Le backend réveille le bot avec un secret à part (`DRIVERSS_KEY_WAKEUP`). Côté serveur uniquement : jamais dans le dashboard ni dans la Mini App |
| Authentification du dashboard | Session Supabase d'un sadran présent dans `dispatchers`, avec un rôle (`dispatcher` ou `admin`). Lectures directes par RLS, limitées à sa station ; actions critiques par les endpoints ci-dessous |
| Authentification des chauffeurs (Mini App) | `Authorization: Bearer <access_token>`, jeton obtenu par J, valable seulement tant que la liaison Telegram qui l'a obtenu est active (D-022). Fonctions préfixées `driver-` |
| Identité de l'acteur | Déduite de l'authentification : session du sadran, jeton du chauffeur, ou `telegram_user_id` reçu par le webhook vérifié du bot |
| Version du contrat | En-tête `x-contract-version: 0` |
| Version des courses | Chaque course porte `version` : 1 à la création, +1 à chaque changement de statut. C'est elle, pas l'heure, qui dit quel état est le plus récent |
| Argent | Entiers en agorot : `price_agorot: 18000` = 180 ₪ |
| Dates | ISO 8601 avec décalage : `2026-10-04T08:00:00+03:00` |
| Téléphones | E.164 : `+9725XXXXXXXX` |
| Idempotence | Les créations acceptent `idempotency_key` : le même appel répété ne crée qu'une seule course. Le claim est idempotent pour un même chauffeur (`replay`). Confirmer deux fois le même envoi (T) ne change rien |
| Succès | `{ "ok": true, ... }` |
| Erreur | `{ "ok": false, "error": { "code": "unknown_place", "message": "..." } }`. `message` sert au débogage : les interfaces affichent leurs propres textes selon `code` |
| Codes HTTP | `200` succès ; `400` requête invalide ; `401` authentification absente ou invalide ; `403` droit insuffisant, chauffeur non lié ou bloqué ; `404` introuvable ; `409` conflit d'état (`already_taken`, `not_claimable`, `version_conflict`, `quote_expired`) ; `429` trop de requêtes |

## Vue d'ensemble

| # | Contrat | Appelant | Servi par | Phase |
| --- | --- | --- | --- | --- |
| A | `POST /lookup-customer` | voice | backend | 2 |
| B | `POST /get-quote` | voice, dispatch (dashboard) | backend | 1 |
| C | `POST /create-ride` | voice, dispatch (dashboard) | backend | 1 |
| D | Contenu des envois « nouvelle course » | dispatch (bot), par la file T | backend | 1 |
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
| R | Contenu des envois « changement de statut » | dispatch (bot), par la file T | backend | 1 |
| S | `POST /manage-driver` | dispatch (dashboard, rôle `admin`) | backend | 1 |
| T | `POST /bot-outbox/pull`, `POST /bot-outbox/ack` | dispatch (bot) | backend | 1 |

## Qui peut appeler quoi

Trois types d'appelants (D-017). Chaque fonction n'accepte que ceux de sa ligne ; l'identité et la station viennent toujours de l'authentification.

| Appelant | Authentification | Station |
| --- | --- | --- |
| Module : `voice`, `dispatch/bot` | `x-driverss-key`, secret du module, côté serveur | Champ `station_id` du corps (Phase 4 : une clé par station) |
| Sadran : dashboard | Session Supabase, utilisateur actif de `dispatchers`, rôle `dispatcher` ou `admin` | Celle du sadran ; un `station_id` différent dans le corps → `forbidden` (403) |
| Chauffeur : Mini App | Jeton obtenu par J, d'une liaison Telegram active | Celle du chauffeur |

| Contrat | Appelants acceptés |
| --- | --- |
| A `lookup-customer` | module `voice` |
| B `get-quote` | module `voice` ; sadran |
| C `create-ride` | module `voice` ; sadran |
| E `claim-ride` | module `dispatch` (bot) |
| F `assign-ride` | sadran |
| G `update-ride-status` | sadran |
| H `GET /rides/{ride_id}` | modules `voice` et `dispatch` ; sadran |
| I webhook de fin d'appel | ElevenLabs, signature vérifiée |
| J `driver-auth-telegram` | personne d'authentifié : la fonction vérifie elle-même l'initData |
| K à O, fonctions `driver-*` | chauffeur |
| P canaux temps réel | chauffeur, sur ses deux canaux seulement |
| Q `link-driver-telegram` | module `dispatch` (bot) |
| S `manage-driver` | sadran avec le rôle `admin` |
| T `bot-outbox` | module `dispatch` (bot) |
| Réveil du bot | le backend vers le bot, secret `DRIVERSS_KEY_WAKEUP` |
| Lectures du dashboard (courses, chauffeurs, historique, signalements) | sadran, directement par RLS, limitées à sa station |

Un `dispatcher` saisit, attribue et clôt les courses ; un `admin` fait la même chose et gère en plus les chauffeurs (S).

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

Calculer un prix et le donner sous forme de devis identifié. Ne crée pas de course.

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
{
  "ok": true,
  "quote_id": "uuid",
  "expires_at": "2026-10-04T07:55:00+03:00",
  "price_agorot": 18000,
  "credit_agorot": 1260,
  "from_place_id": "uuid",
  "to_place_id": "uuid"
}
```

Erreurs : `unknown_place` (lieu non reconnu, avec suggestions), `no_price` (pas de prix pour ce trajet), `closed` (Shabbat ou fête).

Le devis est valable 15 minutes (D-021) : c'est le prix annoncé au client, et c'est lui que `create-ride` enregistrera.

## C. `POST /create-ride`

Créer une course à partir d'un devis accepté par le client (D-021). L'appelant n'envoie jamais de prix : le prix, le trajet, l'horaire, le type et le nombre de passagers viennent du devis.

```json
{
  "idempotency_key": "dash-2026-10-04-0001",
  "station_id": "st_driverss",
  "source": "dashboard",
  "quote_id": "uuid",
  "caller_phone": "+9725XXXXXXXX",
  "institution_code": "1234",
  "pickup_address_text": "...",
  "notes": "",
  "conversation_id": null,
  "replaces_ride_id": null
}
```

```json
{ "ok": true, "ride_id": 153, "status": "created", "version": 1, "price_agorot": 18000 }
```

- `source` vaut `dashboard` ou `voice`. `conversation_id` est rempli par la voix pour la traçabilité.
- `quote_id` est obligatoire. Devis expiré → `quote_expired` (409) : refaire un devis et redemander l'accord du client. Devis inconnu ou d'une autre station → `quote_not_found` (404).
- `replaces_ride_id`, optionnel : la course annulée que celle-ci remplace, après le désistement d'un chauffeur. Sert aux statistiques.
- `station_id` : pour un sadran, ce doit être sa station (sinon `forbidden`) ; pour la voix, c'est la station du numéro appelé.

## D. Envoi « nouvelle course »

Contenu des annonces `ride_posted` (diffusion) et `ride_relaunch` (relance), une par chauffeur destinataire, que le bot lit dans la file des envois (contrat T, D-019). Le contenu est calculé au moment où le bot lit l'envoi. **Jamais de numéro ni de nom de client dedans.**

```json
{
  "ride_id": 153,
  "version": 2,
  "station_id": "st_driverss",
  "from_city": "בני ברק", "from_area": "מרכז",
  "to_city": "ירושלים", "to_area": "רמות",
  "pickup_time": "2026-10-04T08:00:00+03:00",
  "kind": "passengers",
  "passengers": 2,
  "price_agorot": 18000,
  "message_text": "...message au format actuel des chauffeurs, en hébreu..."
}
```

- Destinataires : calculés par le backend (voir « Éligibilité » dans [`ARCHITECTURE.md`](ARCHITECTURE.md)). Le bot n'ajoute ni ne retire personne.
- Relance : annonces `ride_relaunch`, créées par le backend 60 s après la diffusion si la course est toujours libre, avec des destinataires recalculés à ce moment-là, sauf ceux dont la diffusion n'est pas encore partie. La relance verrouille la course comme un changement de statut : un claim simultané passe avant elle (pas de relance) ou après (chaque message de relance confirmé est corrigé). Ensuite, le sadran attribue.
- Chaque message confirmé devient un message connu : quand la course n'est plus libre, le backend le corrige (R, `message_update`).

## E. `POST /claim-ride`

Un chauffeur clique « je prends » dans le bot. Opération atomique : un seul gagnant. Appelant : `dispatch/bot`, avec `x-driverss-key`.

```json
{ "ride_id": 153, "telegram_user_id": 123456789 }
```

`telegram_user_id` est l'auteur du clic, tel que reçu par le webhook Telegram vérifié. Le backend retrouve le chauffeur par la liaison active de ce compte (D-022) ; le bot n'envoie jamais de `driver_id`.

Succès : `{ "ok": true, "replay": false, "ride": { ... } }`, où `ride` est un objet `driver_ride` avec `driver_view: "mine"` et ses `pickup_details` — exemple complet en O.

```json
{ "ok": false, "error": { "code": "already_taken", "message": "ride 153 is already claimed" } }
```

Codes : `already_taken`, `not_claimable` (annulée, terminée, pas encore diffusée), `driver_blocked`, `driver_not_linked` (ce compte Telegram n'a de liaison active avec aucun chauffeur de la station), `not_found`.

Le bot utilise la réponse pour répondre au clic (« הנסיעה שלך! » ou « הנסיעה כבר נלקחה »), et corrige le message cliqué s'il n'est plus juste : c'est un secours, qui ne remplace pas les corrections de la file des envois (T). Le message privé avec les détails part par la file des envois (annonce R `ride_assigned`), quel que soit le canal du claim : une seule voie.

> Changements par rapport à la première v0 : `driver_id` remplacé par `telegram_user_id` (le bot n'avait aucun moyen de connaître `driver_id`) ; erreur au format commun au lieu de `reason` ; réponse identique à O.

## F. `POST /assign-ride`

Le sadran attribue à la main depuis le dashboard. Même fonction atomique que E.

```json
{ "ride_id": 153, "driver_id": "uuid" }
```

Le sadran est identifié par sa session : `assigned_by` n'est plus dans le corps. Réponses : comme O ; le chauffeur n'a pas besoin d'être lié à Telegram. Si un chauffeur a pris la course entre-temps : `already_taken`. Le chauffeur attribué est prévenu par un envoi R (message privé du bot) et par le signal `ride_assigned`.

## G. `POST /update-ride-status`

Clore ou annuler une course.

```json
{ "ride_id": 153, "status": "done", "reason": null, "expected_version": 3 }
```

- `status` vaut `done`, `cancelled` ou `no_show`. L'acteur est déduit de la session : `actor_id` n'est plus dans le corps. Transitions autorisées : voir [`DATABASE.md`](DATABASE.md).
- `expected_version` (recommandé) : la version que le sadran avait à l'écran. Si la course a changé depuis → `version_conflict` (409) : le dashboard relit et le sadran décide. Exemple : il voulait annuler une course libre, mais un chauffeur l'a prise entre-temps.
- Le crédit de 7 % et la commission sont écrits au grand livre uniquement au passage à `done`. L'annulation d'une course déjà prise prévient le chauffeur : avis d'annulation (R `ride_cancelled`), envoyé aussi par précaution si son message d'attribution est peut-être parti, et signal `ride_cancelled`.

## H. `GET /rides/{ride_id}`

Lire l'état d'une course (pour informer le client ou le sadran).

```json
{ "ok": true, "ride_id": 153, "status": "claimed", "version": 3, "driver": { "id": "uuid", "display_name": "..." } }
```

## I. Webhook de fin d'appel (ElevenLabs → backend)

Phase 2. Signature vérifiée. Le backend stocke : `conversation_id`, durée, transcription, résultat (`ride_created`, `transferred`, `failed`), raison d'échec, lien vers l'enregistrement privé.

---

## Mini App chauffeur — conventions (J à P)

- Appelant : `dispatch/miniapp`, sur le téléphone du chauffeur. Aucun secret : seulement l'URL du projet, la clé `anon` publique et le jeton de session.
- Toutes les fonctions sauf J exigent `Authorization: Bearer <access_token>`. Le chauffeur vient du jeton, par sa liaison Telegram active ; la liaison et le statut sont relus à chaque appel (D-022).
- Erreurs communes à K, L, M, N et O : `unauthorized` (401, jeton absent ou expiré), `driver_not_linked` (403, liaison révoquée depuis l'obtention du jeton), `driver_blocked` (403).
- CORS : seule l'origine de la Mini App est acceptée.
- Les textes affichés sont choisis par la Mini App selon `code`, en hébreu ([`MINIAPP.md`](MINIAPP.md)).

### Objet `driver_ride`

La vue d'une course pour un chauffeur. C'est le backend qui décide de ce qui y figure.

```json
{
  "ride_id": 153,
  "driver_view": "available",
  "version": 2,
  "from_city": "בני ברק", "from_area": "מרכז",
  "to_city": "ירושלים", "to_area": "רמות",
  "pickup_time": "2026-10-04T08:00:00+03:00",
  "kind": "passengers",
  "passengers": 2,
  "price_agorot": 18000,
  "message_text": "...message au format actuel des chauffeurs, en hébreu...",
  "posted_at": "2026-10-04T07:41:12+03:00",
  "pickup_details": null
}
```

| `driver_view` | Sens | Champs présents |
| --- | --- | --- |
| `available` | Course `posted` que ce chauffeur peut prendre | Tous, avec `pickup_details: null` |
| `mine` | Course `claimed` par ce chauffeur | Tous, avec `pickup_details` |
| `taken` | Prise par un autre chauffeur | `ride_id`, `driver_view` et `version` seulement |
| `closed` | Annulée, terminée ou client absent | `ride_id`, `driver_view` et `version` seulement |

`version` est présente dans toutes les vues : la Mini App ignore une réponse dont la version est plus ancienne que celle qu'elle affiche déjà.

`pickup_details`, seulement pour `mine` :

```json
{ "address_text": "...", "customer_phone": "+9725XXXXXXXX", "notes": "..." }
```

## J. `POST /driver-auth-telegram`

Échanger l'initData de Telegram contre une session. Appelée une fois à chaque ouverture de la Mini App, sans jeton (seulement la clé `anon` du projet). La session vaut pour la liaison Telegram active du chauffeur (D-022) : si cette liaison est révoquée, le jeton est refusé partout dès l'appel suivant, même avant son expiration.

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
| `driver_not_linked` | 403 | Aucune liaison active entre ce compte Telegram et un chauffeur de la station (jamais lié, ou délié) |
| `driver_blocked` | 403 | Chauffeur bloqué |

Idempotence : chaque appel crée une nouvelle session ; l'appeler deux fois est sans risque. Seul effet en base : la création de l'identité Supabase de la liaison active, une seule fois, à la première connexion qui suit la liaison.

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
- Effet : seulement sur les destinataires des prochains envois (D). Ne retire ni une course déjà prise, ni un message déjà envoyé.
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
      "version": 2,
      "from_city": "בני ברק", "from_area": "מרכז",
      "to_city": "ירושלים", "to_area": "רמות",
      "pickup_time": "2026-10-04T08:00:00+03:00",
      "kind": "passengers",
      "passengers": 2,
      "price_agorot": 18000,
      "message_text": "...",
      "posted_at": "2026-10-04T07:41:12+03:00",
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
{ "ok": true, "server_time": "2026-10-04T07:43:05+03:00", "ride": { "ride_id": 153, "driver_view": "taken", "version": 3 } }
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
    "version": 3,
    "from_city": "בני ברק", "from_area": "מרכז",
    "to_city": "ירושלים", "to_area": "רמות",
    "pickup_time": "2026-10-04T08:00:00+03:00",
    "kind": "passengers",
    "passengers": 2,
    "price_agorot": 18000,
    "message_text": "...",
    "posted_at": "2026-10-04T07:41:12+03:00",
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
- Après un succès : signaux P, et envois R dans la file du bot.

## P. Canaux temps réel chauffeur

Servis par le backend (Supabase Realtime, Broadcast). La Mini App s'abonne avec supabase-js, en canal privé (`config: { private: true }`), avec son jeton de session.

| Canal | Qui peut écouter | Événement | Charge utile |
| --- | --- | --- | --- |
| `rides:{station_id}` | Chauffeurs `active` de la station | `ride_available` | `{ "ride_id": 153, "version": 2, "at": "2026-10-04T07:41:12+03:00" }` |
| `rides:{station_id}` | Idem | `ride_unavailable` | `{ "ride_id": 153, "version": 3, "reason": "claimed", "at": "2026-10-04T07:42:03+03:00" }` — `reason` : `claimed` ou `cancelled` |
| `driver:{driver_id}` | Ce chauffeur | `ride_assigned` | `{ "ride_id": 153, "version": 3, "via": "dashboard", "at": "2026-10-04T07:42:03+03:00" }` — `via` : `telegram_bot`, `miniapp` ou `dashboard` |
| `driver:{driver_id}` | Ce chauffeur | `ride_cancelled` | `{ "ride_id": 153, "version": 4, "at": "2026-10-04T07:50:00+03:00" }` |
| `driver:{driver_id}` | Ce chauffeur | `ride_closed` | `{ "ride_id": 153, "version": 4, "status": "done", "at": "2026-10-04T09:10:00+03:00" }` — `status` : `done` ou `no_show` |

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
| `telegram_already_linked` | Ce chauffeur a déjà une liaison active avec un autre compte Telegram, ou ce compte est lié à un autre chauffeur : un `admin` doit délier (S) |
| `link_requires_admin` | Le chauffeur a été délié avec la raison `compromised` (téléphone perdu ou volé, compte compromis, D-022). Une nouvelle liaison n'est possible que pendant l'autorisation d'un `admin` (S, `allow_relink`), et avec l'ancien compte Telegram seulement si l'admin l'a accepté. Le bot dit d'appeler le sadran |
| `driver_blocked` | Chauffeur bloqué |

- Chaque liaison réussie est une nouvelle ligne de `driver_links`, qui recevra sa propre identité Supabase à la première connexion de la Mini App (J). Le `driver_id`, les courses et l'historique comptable ne changent pas.
- Idempotence : la même paire deux fois, tant que la liaison est active → `ok: true` avec `already_linked: true`. Chaque liaison est journalisée (`driver_events`).

## R. Envois « changement de statut »

Contenu des envois créés quand une course change de statut après `posted`, quel que soit le canal d'origine (bot, Mini App, dashboard). Le bot les lit dans la file des envois (contrat T, D-019). Deux familles :

- **annonce** : un nouveau message qui apporte une nouveauté, ici l'attribution ; abandonnée si la course change avant que le bot la lise ;
- **correction** : remet à jour ce que le chauffeur a déjà reçu ; jamais abandonnée parce que la course a changé, son contenu suit l'état actuel.

| Envoi (`kind`) | Famille | À qui | Ce que fait le bot |
| --- | --- | --- | --- |
| `ride_assigned` | Annonce | Le chauffeur attribué, s'il a une liaison active | Nouveau message privé avec les détails de prise en charge. Seul envoi qui contient `pickup_details` |
| `message_update` | Correction | Chaque message connu de la course dont l'état affiché n'est plus le bon : diffusion, relance et attribution, chez tous les destinataires, gagnant compris, quel que soit le canal du claim ou de l'attribution | Modifie le message `edit_message_id` selon `state`, sans bouton « אני לוקח » ni détails de prise en charge |
| `ride_cancelled` | Correction | Le chauffeur attribué, si son message d'attribution est parti ou peut-être parti (T) | Nouveau message « הנסיעה בוטלה », qui rappelle le trajet et l'heure |

`state` : l'état qu'un message doit afficher, calculé par le backend au moment où le bot lit l'envoi. Le texte de chaque état est choisi par le bot, en hébreu.

| Message | Course `posted` | `claimed`, `done`, `no_show` | `cancelled` |
| --- | --- | --- | --- |
| Diffusion ou relance, chez le chauffeur attribué | `available` | `yours` | `cancelled` |
| Diffusion ou relance, chez un autre chauffeur | `available` | `taken` | `cancelled` |
| Attribution | — | `yours` | `cancelled` |

Un message n'est corrigé que si l'état qu'il doit afficher change : `done` et `no_show` ne créent donc aucun envoi.

`ride_assigned` :

```json
{
  "ride_id": 153,
  "version": 3,
  "station_id": "st_driverss",
  "via": "miniapp",
  "message_text": "...message au format actuel des chauffeurs, en hébreu...",
  "pickup_details": { "address_text": "...", "customer_phone": "+9725XXXXXXXX", "notes": "..." }
}
```

`message_update`, et `ride_cancelled` sans `state` :

```json
{
  "ride_id": 153,
  "version": 4,
  "station_id": "st_driverss",
  "state": "cancelled",
  "message_text": "...message au format actuel des chauffeurs, en hébreu..."
}
```

- `via` : canal du claim ou de l'attribution (`telegram_bot`, `miniapp`, `dashboard`).
- Un message modifié ne contient jamais de détails de prise en charge : une attribution annulée perd l'adresse et le téléphone du client.
- Rien n'est stocké dans la file : le contenu, détails de prise en charge compris, est lu dans la course au moment où le bot lit l'envoi.
- Un chauffeur sans liaison active ne reçoit aucun envoi : le sadran l'appelle. Si sa course est annulée, le dashboard le signale (« Envois du bot : fiabilité » dans [`ARCHITECTURE.md`](ARCHITECTURE.md)).

## S. `POST /manage-driver`

Gérer les chauffeurs depuis le dashboard (D-017). Appelant : sadran avec le rôle `admin`. La station vient de sa session.

```json
{ "action": "create", "phone_e164": "+9725XXXXXXXX", "display_name": "..." }
```

```json
{ "action": "block", "driver_id": "uuid", "reason": "..." }
```

```json
{ "action": "unlink_telegram", "driver_id": "uuid", "reason": "compromised" }
```

```json
{ "action": "allow_relink", "driver_id": "uuid", "allow_same_account": false }
```

Actions : `create`, `update` (`display_name`, `phone_e164`), `block`, `unblock`, `unlink_telegram`, `allow_relink`.

```json
{
  "ok": true,
  "driver": {
    "driver_id": "uuid",
    "display_name": "...",
    "phone_e164": "+9725XXXXXXXX",
    "status": "active",
    "telegram_linked": false,
    "relink": { "requires_admin": true, "allowed_until": null, "same_account_allowed": false }
  }
}
```

- `block` met `status = blocked` et révoque les sessions du chauffeur (D-011). Effet immédiat : chaque appel relit le statut.
- `unlink_telegram` (D-022), avec `reason` : `changed_account` (le chauffeur a changé de compte Telegram) ou `compromised` (téléphone perdu ou volé, compte compromis). Dans cet ordre :
  1. en base, dans une transaction : la liaison passe à `revoked` ; les envois en attente vers elle sont abandonnés ; les courses en cours du chauffeur sont signalées au sadran ; avec `compromised`, le chauffeur ne peut plus se relier seul (Q) ;
  2. ensuite, le backend supprime l'identité Supabase de cette liaison, et réessaie jusqu'à réussir.

  L'ancien jeton est refusé dès l'étape 1, même avant son expiration et même si l'étape 2 échoue. Le `driver_id`, les courses et l'historique comptable restent.
- `allow_relink`, après une déliaison `compromised` : autorise une nouvelle liaison par Q pendant 15 minutes. L'admin le fait au téléphone avec le chauffeur. `allow_same_account: true` accepte l'ancien compte Telegram, seulement si le chauffeur l'a récupéré et a fermé les autres sessions (Telegram : Paramètres → Appareils). Une fois la nouvelle liaison faite, le verrou posé par `compromised` est levé.
- Erreurs : `forbidden` (403, pas `admin`), `driver_not_found` (404), `phone_already_used` (409).
- Chaque action est journalisée (`driver_events`) : qui, quoi, quand, et la raison.
- Idempotent : bloquer un chauffeur déjà bloqué, débloquer un chauffeur actif ou délier un chauffeur sans liaison active ne change rien.

## T. File des envois du bot — `POST /bot-outbox/pull` et `POST /bot-outbox/ack`

Proposition D-019. Appelant : `dispatch/bot`, avec `x-driverss-key`. Le backend dépose les envois dans une file durable, dans la même transaction que le changement qui les rend nécessaires ; le bot les lit, les envoie et confirme chacun. Règles complètes et cas d'échec : « Envois du bot : fiabilité » dans [`ARCHITECTURE.md`](ARCHITECTURE.md).

| Famille | Envois | Si la course change avant que le bot lise l'envoi |
| --- | --- | --- |
| Annonce : un nouveau message qui apporte une nouveauté | `ride_posted`, `ride_relaunch` (D), `ride_assigned` (R) | Abandonnée (`skipped`) |
| Correction : remettre à jour ce que le chauffeur a déjà reçu | `message_update`, `ride_cancelled` (R) | Gardée ; son contenu suit l'état actuel |

**Réveil :** quand des envois sont prêts, le backend appelle le webhook du bot, sans données, avec le secret `DRIVERSS_KEY_WAKEUP` dans `x-driverss-key`. Il le refait régulièrement par sécurité (par exemple toutes les 15 s). Un réveil perdu ne fait que retarder l'envoi.

**Lire** — `POST /bot-outbox/pull` :

```json
{ "max": 20 }
```

```json
{
  "ok": true,
  "jobs": [
    {
      "job_id": "uuid",
      "kind": "ride_posted",
      "action": "send",
      "ride_id": 153,
      "version": 2,
      "attempt": 1,
      "send_before": "2026-10-04T07:41:57+03:00",
      "recipient": { "driver_id": "uuid", "telegram_user_id": 123456789 },
      "edit_message_id": null,
      "payload": { "ride_id": 153, "version": 2, "message_text": "..." }
    },
    {
      "job_id": "uuid",
      "kind": "message_update",
      "action": "edit",
      "ride_id": 150,
      "version": 4,
      "attempt": 2,
      "send_before": "2026-10-04T07:41:57+03:00",
      "recipient": { "driver_id": "uuid", "telegram_user_id": 123456789 },
      "edit_message_id": 4567,
      "payload": { "ride_id": 150, "version": 4, "station_id": "st_driverss", "state": "taken", "message_text": "..." }
    }
  ]
}
```

- `action` : `send` (nouveau message) ou `edit` (modifier le message connu `edit_message_id`).
- `payload` : D pour `ride_posted` et `ride_relaunch`, R pour les autres. Il est calculé au moment de la lecture : la file ne stocke aucun contenu.
- Réservation : un envoi lu est réservé 60 s. Le bot ne l'envoie plus après `send_before` (45 s après la lecture), et ses appels à Telegram durent 10 s au plus : aucune tentative n'arrive chez Telegram après la fin de sa réservation. Un envoi que le bot n'a pas tenté avant `send_before` est rendu par `retry`, avec `retry_after_s: 0`.
- Une annonce n'est pas remise si la course a changé de `version` depuis sa création, ou si la liaison du destinataire est révoquée. Une diffusion ou une relance ne l'est pas non plus si le chauffeur est bloqué ou si l'heure de prise en charge est passée. Une attribution reste remise même si l'heure est passée : le chauffeur a besoin de ses détails.
- Une correction n'est jamais abandonnée parce que la version a changé. Une seule `message_update` en cours par message ; elle n'est pas remise si le message affiche déjà l'état actuel (sans objet). `ride_cancelled` attend la fin de la tentative d'attribution en cours ; elle part si l'attribution est partie ou peut-être partie, et seulement dans ce cas.
- Pendant une fermeture (Shabbat, fête), `pull` ne renvoie rien (D-020).

**Confirmer** — `POST /bot-outbox/ack`, juste après la réponse de Telegram :

```json
{ "job_id": "uuid", "attempt": 1, "result": "sent", "telegram_message_id": 4567 }
```

```json
{ "job_id": "uuid", "attempt": 1, "result": "retry", "retry_after_s": 12 }
```

```json
{ "job_id": "uuid", "attempt": 1, "result": "failed", "error": "bot_blocked" }
```

```json
{ "job_id": "uuid", "attempt": 1, "result": "unknown" }
```

Réponse : `{ "ok": true }`.

| `result` | Quand | Ce que fait le backend |
| --- | --- | --- |
| `sent` | Telegram a accepté ; aussi quand il répond que le message modifié est déjà identique | Enregistre le message connu, ou son nouvel état, puis le compare à l'état actuel de la course : s'ils diffèrent, une correction suit |
| `retry` | Telegram a refusé pour l'instant (429, avec un délai) : rien n'est parti | Remet l'envoi en file après `retry_after_s` |
| `failed` | Refus définitif de Telegram : `bot_blocked` (le chauffeur a bloqué le bot), `chat_not_found`, `message_unavailable` (message à modifier introuvable ou plus modifiable) | Échec ; signalement au sadran quand c'est nécessaire |
| `unknown` | Pas de réponse claire : délai dépassé, connexion coupée, erreur 5xx | Marque l'envoi « peut-être parti », puis le remet en file |

- Une absence de confirmation ne prouve jamais une absence d'envoi : une réservation expirée sans réponse compte comme `unknown`, et un envoi « peut-être parti » le reste pour toujours. Le bot ne répond `retry` ou `failed` que si Telegram a clairement refusé.
- Après 5 tentatives sans `sent`, l'envoi passe en échec.
- Confirmation tardive (après la fin de la réservation) : pour un nouveau message, `sent` est toujours enregistré, puisque le message existe, même si une autre tentative a été lue entre-temps. Les autres réponses d'une tentative dépassée sont ignorées : le backend recompare et corrige si besoin.
- Confirmer deux fois la même tentative ne change rien.
- Limites de Telegram : moins de 30 messages par seconde au total, et au plus un par seconde au même chauffeur, modifications comprises.
- Doublon possible : un envoi peut-être parti est renvoyé, donc un chauffeur peut exceptionnellement recevoir deux fois le même message. Accepté ; chaque copie confirmée est corrigée comme les autres.

---

## Ce qui reste interne à un module

- Le webhook Telegram (Telegram → `dispatch/bot`) est interne au dispatch. Il vérifie l'en-tête secret envoyé par Telegram, puis appelle E ou Q.
- Les textes et boutons des messages du bot, et les écrans de la Mini App et du dashboard, sont internes à `dispatch/`.
- Le prompt et la configuration de l'agent vocal sont internes à `voice/`.
