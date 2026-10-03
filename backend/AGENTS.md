# backend/ — AGENTS.md

**Responsable : Oren.** Lis d'abord le `AGENTS.md` à la racine.

## Mission

Le cœur de Driverss : la source de vérité. Toute décision métier critique, toute donnée persistée et toute transition de statut passent par ce module.

## Stack

- Supabase : Postgres, RLS, Realtime, Storage (bucket privé pour les enregistrements), Edge Functions (TypeScript, Deno).
- Supabase CLI pour les migrations et le déploiement.

## Organisation

```
backend/
└── supabase/
    ├── migrations/        ← une migration par changement de schéma, jamais modifiée après fusion
    ├── functions/
    │   ├── _shared/       ← auth entre modules, validation, erreurs, helpers
    │   ├── create-ride/
    │   ├── get-quote/
    │   ├── claim-ride/
    │   ├── assign-ride/
    │   ├── update-ride-status/
    │   └── ...
    └── seed.sql           ← données de test fictives uniquement
```

Les dossiers des fonctions sont créés en Phase 1, pas avant la porte G0.

## Règles

- Chaque endpoint suit exactement `docs/API_CONTRACTS.md`. Changer un contrat = modifier ce fichier dans la même pull request.
- Chaque table métier porte `station_id`. RLS activé partout.
- Argent en agorot (entiers). Dates en `timestamptz`.
- Les changements de statut passent par des fonctions SQL atomiques. Le claim est une seule requête conditionnelle (`posted` → `claimed`) ; zéro ligne modifiée = `already_taken`.
- Le grand livre (`ledger_entries`) est en ajout seul. Le crédit de 7 % et la commission sont écrits au passage à `done`, dans la même transaction.
- Chaque appel entrant vérifie `x-driverss-key`. Chaque webhook vérifie sa signature.
- La clé `service_role` ne sort jamais du serveur.
- `seed.sql` ne contient que des données inventées. Jamais de vraies données de Yossef.
- Logs sans secrets ni numéros de téléphone complets.

## Tests obligatoires avant fusion

- Tests unitaires et d'API de chaque endpoint.
- Mauvais secret → refus.
- Ville inconnue → `unknown_place`.
- Deux claims simultanés → un seul gagnant.
- Même `idempotency_key` deux fois → une seule course.
