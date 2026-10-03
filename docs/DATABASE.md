# Base de données

> **Version 0 — brouillon.** Schéma à finaliser en Phase 0, après analyse de l'export du logiciel de Yossef. Responsable : Oren.

Base : Postgres (Supabase). Toute modification passe par une migration dans `backend/supabase/migrations/`.

## Règles

1. **`station_id` dans chaque table métier**, dès le premier jour.
2. **Argent en agorot** (entiers). Jamais de `float` ni de `numeric` à virgule pour l'argent.
3. **Dates en `timestamptz`**, stockées en UTC.
4. **Statuts modifiés uniquement par des fonctions backend** (SQL ou Edge Functions), jamais par un `UPDATE` venu d'un frontend ou d'un bot.
5. **Le grand livre est en ajout seul.** On ne modifie ni ne supprime une ligne : on corrige par une écriture inverse.
6. **RLS activé sur toutes les tables.** Le dashboard ne voit que les données de sa station.
7. **Pas de données personnelles inutiles.** Les enregistrements d'appels vont dans un bucket privé, avec une durée de conservation définie.

## Tables (v0)

| Table | Rôle | Colonnes principales |
| --- | --- | --- |
| `stations` | Une station cliente (Driverss d'abord) | `id`, `name`, `brand_name`, `timezone`, `created_at` |
| `customers` | Clients inscrits | `id`, `station_id`, `phone_e164`, `name`, `institution_id`, `consent_at`, `created_at` |
| `institutions` | Synagogues et associations (codes Nedarim Plus) | `id`, `station_id`, `code`, `name` |
| `places` | Villes et quartiers reconnus | `id`, `station_id`, `city`, `area` |
| `place_aliases` | Variantes de noms (prononciation, yiddish, abréviations) | `id`, `place_id`, `alias` |
| `prices` | Table de prix (si validée en Phase 0) | `id`, `station_id`, `from_place_id`, `to_place_id`, `kind`, `price_agorot` |
| `drivers` | Chauffeurs | `id`, `station_id`, `phone_e164`, `display_name`, `telegram_user_id`, `status` (`active`, `blocked`), `created_at` |
| `rides` | Courses | `id`, `station_id`, `customer_id`, `source`, `from_place_id`, `to_place_id`, `pickup_address_text`, `pickup_time`, `kind`, `passengers`, `price_agorot`, `credit_agorot`, `institution_id`, `status`, `driver_id`, `claimed_at`, `idempotency_key`, `conversation_id`, `created_at` |
| `ride_events` | Historique de chaque course | `id`, `ride_id`, `type`, `actor_type`, `actor_id`, `payload`, `created_at` |
| `ledger_entries` | Grand livre : crédits de 7 %, commissions, dons | `id`, `station_id`, `ride_id`, `account` (`customer_credit`, `institution_donation`, `driver_commission`), `account_owner_id`, `amount_agorot`, `created_at` |
| `calls` | Appels traités par la voix (Phase 2) | `id`, `station_id`, `conversation_id`, `caller_phone`, `duration_s`, `outcome`, `failure_reason`, `transcript`, `recording_path`, `created_at` |
| `closures` | Fermetures (Shabbat, fêtes) | `id`, `station_id`, `starts_at`, `ends_at`, `reason` |

## Statuts d'une course

| De | Vers | Déclenché par |
| --- | --- | --- |
| — | `created` | `create-ride` |
| `created` | `posted` | Backend, dès que la course est envoyée au dispatch |
| `posted` | `claimed` | `claim-ride` ou `assign-ride` (atomique) |
| `claimed` | `done` | `update-ride-status` → écrit le crédit de 7 % et la commission |
| `created`, `posted`, `claimed` | `cancelled` | `update-ride-status` → rien au grand livre |
| `claimed` | `no_show` | `update-ride-status` → rien au grand livre (à valider avec Yossef) |

Toute autre transition est refusée par le backend.

## Claim atomique

Le claim est une seule requête conditionnelle : la course passe à `claimed` seulement si elle est encore `posted`. Si zéro ligne est modifiée, la réponse est `already_taken`. Test obligatoire : deux claims simultanés, un seul gagnant.

## Questions ouvertes (Phase 0)

- Source de vérité des soldes de 7 % : le logiciel de Yossef ou cette base ?
- Les prix viennent-ils d'une table fixe ou d'une négociation dans le groupe ?
- Quelles données exactes contient l'export du logiciel ?
- Durée de conservation des enregistrements d'appels (avis de l'avocat).
