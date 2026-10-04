# backend/ — AGENTS.md

**Responsable : Oren.** Lis d'abord le `AGENTS.md` à la racine.

## Mission

Le cœur de Driverss : la source de vérité. Toute décision métier critique, toute donnée persistée et toute transition de statut passent par ce module. Il authentifie aussi les chauffeurs de la Mini App et décide de l'éligibilité.

## Stack

- Supabase : Postgres, RLS, Auth (sessions des sadranim et des chauffeurs), Realtime (Broadcast privé), Storage (bucket privé pour les enregistrements), Edge Functions (TypeScript, Deno).
- Supabase CLI pour les migrations et le déploiement.

## Organisation

```
backend/
└── supabase/
    ├── migrations/        ← une migration par changement de schéma, jamais modifiée après fusion
    ├── functions/
    │   ├── _shared/                 ← auth (module, chauffeur), validation, erreurs, helpers
    │   ├── create-ride/
    │   ├── get-quote/
    │   ├── claim-ride/              ← bot
    │   ├── assign-ride/
    │   ├── update-ride-status/
    │   ├── link-driver-telegram/    ← bot : lier un compte Telegram à un chauffeur pré-inscrit
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
- Les changements de statut passent par des fonctions SQL atomiques. Le claim et l'attribution passent par une seule fonction, `claim_ride` (esquisse dans `docs/DATABASE.md`) : une requête conditionnelle (`posted` → `claimed`) ; zéro ligne modifiée = refus expliqué (`already_taken`, `replay` pour le même chauffeur…). Elle n'est pas exécutable par `anon` ni `authenticated`.
- Le grand livre (`ledger_entries`) est en ajout seul. Le crédit de 7 % et la commission sont écrits au passage à `done`, dans la même transaction.
- Deux familles d'appelants, jamais mélangées dans une même fonction : les modules (`x-driverss-key`) et les chauffeurs (`Authorization: Bearer`, fonctions `driver-*`). Chaque webhook vérifie sa signature.
- L'initData Telegram n'est validé que dans `driver-auth-telegram` (signature, `auth_date` de moins de 300 s). Ensuite, le chauffeur vient du jeton (`auth_user_id`) et son statut est relu à chaque appel. Jamais d'identité prise dans le corps de la requête.
- L'éligibilité se calcule ici seulement : destinataires de D, liste `driver-rides`, claim.
- Les chauffeurs n'ont aucune politique RLS sur les tables métier. Leur seule politique : la réception de leurs canaux sur `realtime.messages`.
- Chaque changement de statut écrit `ride_events` et émet un signal Broadcast sans donnée personnelle, dans la même transaction. Les événements D et R partent de `ride_events` : au moins une fois, avec `event_id`.
- La clé `service_role` ne sort jamais du serveur.
- `seed.sql` ne contient que des données inventées. Jamais de vraies données de Yossef.
- Logs sans secrets, sans initData, sans jetons ni numéros de téléphone complets.

## Tests obligatoires avant fusion

- Tests unitaires et d'API de chaque endpoint.
- Mauvais secret → refus.
- Ville inconnue → `unknown_place`.
- Deux claims simultanés → un seul gagnant, y compris un par le bot et un par la Mini App ; 20 claims parallèles → 1 `claimed`, 19 `already_taken`, 20 lignes dans `ride_events`.
- Même chauffeur deux fois → `replay`, une seule ligne `claimed`.
- Même `idempotency_key` deux fois → une seule course.
- initData falsifié, trop vieux ou d'un autre bot → refus.
- Jeton de chauffeur : lecture directe de `rides` ou `customers` → zéro ligne ; appel RPC de `claim_ride` → refus.
- Chauffeur bloqué → refus immédiat sur toutes les fonctions `driver-*`.
- Liste `available`, signaux temps réel et événement D : aucun champ client.
