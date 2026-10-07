# backend/ — AGENTS.md

**Responsable : Oren.** Lis d'abord le `AGENTS.md` à la racine.

## Mission

Le cœur de Driverss : la source de vérité. Toute décision métier critique, toute donnée persistée et toute transition de statut passent par ce module. Il authentifie aussi les chauffeurs de la Mini App, décide de l'éligibilité et des droits de chaque appelant, et tient la file des envois du bot.

## Stack

- Supabase : Postgres, RLS, Auth (sessions des sadranim et des chauffeurs), Realtime (Broadcast privé), Storage (bucket privé pour les enregistrements), Edge Functions (TypeScript, Deno).
- Supabase CLI pour les migrations et le déploiement.

## Organisation

```
backend/
└── supabase/
    ├── migrations/        ← une migration par changement de schéma, jamais modifiée après fusion
    ├── functions/
    │   ├── _shared/                 ← auth (modules, sadranim, chauffeurs), droits, validation, erreurs, helpers
    │   ├── create-ride/
    │   ├── get-quote/
    │   ├── claim-ride/              ← bot
    │   ├── assign-ride/
    │   ├── update-ride-status/
    │   ├── manage-driver/           ← dashboard (rôle admin) : créer, bloquer, délier un chauffeur
    │   ├── link-driver-telegram/    ← bot : lier un compte Telegram à un chauffeur pré-inscrit
    │   ├── bot-outbox/              ← bot : lire et confirmer les envois de la file
    │   ├── driver-auth-telegram/    ← Mini App : initData → session
    │   ├── driver-me/
    │   ├── driver-rides/
    │   ├── driver-claim-ride/
    │   ├── driver-set-availability/
    │   └── ...
    └── seed.sql           ← données de test fictives uniquement
```

Les dossiers des fonctions sont créés en Phase 1, pas avant la porte G0.

## Règles

- Chaque endpoint suit exactement `docs/API_CONTRACTS.md`. Changer un contrat = modifier ce fichier dans la même pull request.
- Chaque table métier porte `station_id`. RLS activé partout.
- Argent en agorot (entiers). Dates en `timestamptz`.
- Trois types d'appelants : modules (`x-driverss-key`), sadranim (session, table `dispatchers`, rôle `dispatcher` ou `admin`) et chauffeurs (jeton du contrat J). Chaque fonction n'accepte que ceux de sa ligne dans le tableau « Qui peut appeler quoi » de `docs/API_CONTRACTS.md` ; l'identité et la station viennent toujours de l'authentification, par la fonction commune de `_shared/`. Chaque webhook vérifie sa signature.
- L'initData Telegram n'est validé que dans `driver-auth-telegram` (signature, `auth_date` de moins de 300 s). Ensuite, le chauffeur vient du jeton (`auth_user_id`) et son statut est relu à chaque appel. Jamais d'identité prise dans le corps de la requête.
- Les changements de statut passent par des fonctions SQL atomiques. Le claim et l'attribution passent par une seule fonction, `claim_ride` (esquisse dans `docs/DATABASE.md`) : une requête conditionnelle (`posted` → `claimed`) ; zéro ligne modifiée = refus expliqué (`already_taken`, `replay` pour le même chauffeur…). Elle vit dans le schéma `private`, jamais exposé par l'API, et n'est pas exécutable par `anon` ni `authenticated`.
- Chaque changement de statut fait, dans la même transaction : `version` + 1, une ligne `ride_events`, un signal Broadcast sans donnée personnelle, et les envois du bot dans `notifications` (D-019). Un envoi dépassé (version plus ancienne que celle de la course) n'est jamais remis au bot. `update-ride-status` respecte `expected_version` quand il est fourni.
- `create-ride` n'accepte qu'un devis valide (`quote_id`, D-021) : le prix enregistré est celui du devis.
- Le grand livre (`ledger_entries`) est en ajout seul. Le crédit de 7 % et la commission sont écrits au passage à `done`, dans la même transaction, avec la règle d'arrondi de D-018 et une seule écriture par course et par compte.
- L'éligibilité se calcule ici seulement : envois D et relance, liste `driver-rides`, claim.
- La relance à 60 s est une tâche planifiée du backend, qui recalcule les destinataires. Pendant une fermeture (`closures`), aucun envoi ni relance (D-020).
- Les chauffeurs n'ont aucune politique RLS sur les tables métier. Leur seule politique : la réception de leurs canaux sur `realtime.messages`, par la fonction d'aide `private.driver_can_listen`.
- La clé `service_role` ne sort jamais du serveur.
- MCP Supabase (plugin `supabase`) : projet de développement uniquement, jamais la production ; lecture seule, limité à ce projet. Ne jamais l'utiliser pour modifier le schéma ou les données : tout changement passe par une migration dans `backend/supabase/migrations/`.
- `seed.sql` ne contient que des données inventées. Jamais de vraies données de Yossef.
- Logs sans secrets, sans initData, sans jetons ni numéros de téléphone complets.

## Tests obligatoires avant fusion

- Tests unitaires et d'API de chaque endpoint.
- Mauvais secret → refus.
- Tableau des droits : chaque fonction refuse les appelants qui ne sont pas sur sa ligne ; un sadran d'une autre station → refus ; `manage-driver` sans le rôle `admin` → refus.
- Ville inconnue → `unknown_place`.
- Devis expiré → `quote_expired` ; le prix enregistré est celui du devis, même si la table de prix a changé entre-temps.
- Deux claims simultanés → un seul gagnant, y compris un par le bot et un par la Mini App ; 20 claims parallèles → 1 `claimed`, 19 `already_taken`, 20 lignes dans `ride_events`.
- Même chauffeur deux fois → `replay`, une seule ligne `claimed`.
- Claim puis annulation : la course est prise puis annulée, et le chauffeur est prévenu ; annulation avec une `expected_version` périmée → `version_conflict`, course inchangée.
- Même `idempotency_key` deux fois → une seule course.
- Passage à `done` rejoué → une seule écriture par compte au grand livre.
- File des envois : une course annulée avant l'envoi au gagnant → l'envoi est abandonné ; un envoi non confirmé revient après 60 s ; aucun envoi pendant une fermeture.
- initData falsifié, trop vieux ou d'un autre bot → refus.
- Jeton de chauffeur : lecture directe de `rides` ou `customers` → zéro ligne ; appel RPC de `claim_ride` → refus.
- Temps réel : un chauffeur actif peut s'abonner à ses deux canaux, pas à ceux d'un autre chauffeur ni d'une autre station.
- Chauffeur bloqué → refus immédiat sur toutes les fonctions `driver-*`.
- Liste `available`, signaux temps réel et envois D : aucun champ client.
