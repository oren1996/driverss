# Base de données

> **Version 0 — brouillon.** Schéma à finaliser en Phase 0, après analyse de l'export du logiciel de Yossef. Responsable : Oren. Mis à jour le 4 octobre 2026 pour la Mini App chauffeur : voir « Changements pour la Mini App ».

Base : Postgres (Supabase). Toute modification passe par une migration dans `backend/supabase/migrations/`.

## Règles

1. **`station_id` dans chaque table métier**, dès le premier jour.
2. **Argent en agorot** (entiers). Jamais de `float` ni de `numeric` à virgule pour l'argent.
3. **Dates en `timestamptz`**, stockées en UTC.
4. **Statuts modifiés uniquement par des fonctions backend** (SQL ou Edge Functions), jamais par un `UPDATE` venu d'un frontend ou d'un bot.
5. **Le grand livre est en ajout seul.** On ne modifie ni ne supprime une ligne : on corrige par une écriture inverse.
6. **RLS activé sur toutes les tables.** Le dashboard ne voit que les données de sa station.
7. **Pas de données personnelles inutiles.** Les enregistrements d'appels vont dans un bucket privé, avec une durée de conservation définie.
8. **Les chauffeurs n'accèdent à aucune table métier.** Ils passent par les fonctions `driver-*`. Leur seule politique RLS porte sur la réception temps réel (`realtime.messages`).
9. **Une politique RLS ne repose jamais sur le seul rôle `authenticated`.** Chauffeurs et sadranim sont tous deux `authenticated` : la politique vérifie l'appartenance (sadran de la station, `drivers.auth_user_id`).
10. **Les fonctions `security definer` ne sont pas exécutables par `anon` ni `authenticated`** : schéma non exposé par l'API, ou `revoke execute`.

## Tables (v0)

| Table | Rôle | Colonnes principales |
| --- | --- | --- |
| `stations` | Une station cliente (Driverss d'abord) | `id`, `name`, `brand_name`, `timezone`, `driver_support_phone_e164`, `created_at` |
| `customers` | Clients inscrits | `id`, `station_id`, `phone_e164`, `name`, `institution_id`, `consent_at`, `created_at` |
| `institutions` | Synagogues et associations (codes Nedarim Plus) | `id`, `station_id`, `code`, `name` |
| `places` | Villes et quartiers reconnus | `id`, `station_id`, `city`, `area` |
| `place_aliases` | Variantes de noms (prononciation, yiddish, abréviations) | `id`, `station_id`, `place_id`, `alias` |
| `prices` | Table de prix (si validée en Phase 0) | `id`, `station_id`, `from_place_id`, `to_place_id`, `kind`, `price_agorot` |
| `drivers` | Chauffeurs | `id`, `station_id`, `phone_e164`, `display_name`, `telegram_user_id` (`bigint`), `telegram_linked_at`, `auth_user_id`, `status` (`active`, `blocked`), `is_available`, `available_until`, `created_at` |
| `rides` | Courses | `id`, `station_id`, `customer_id`, `source`, `from_place_id`, `to_place_id`, `pickup_address_text`, `pickup_time`, `kind`, `passengers`, `price_agorot`, `credit_agorot`, `institution_id`, `notes`, `status`, `driver_id`, `posted_at`, `claimed_at`, `updated_at`, `idempotency_key`, `conversation_id`, `created_at` |
| `ride_events` | Historique de chaque course ; source des événements envoyés à `dispatch/` | `id`, `station_id`, `ride_id`, `type`, `actor_type`, `actor_id`, `payload`, `created_at` |
| `ledger_entries` | Grand livre : crédits de 7 %, commissions, dons | `id`, `station_id`, `ride_id`, `account` (`customer_credit`, `institution_donation`, `driver_commission`), `account_owner_id`, `amount_agorot`, `created_at` |
| `calls` | Appels traités par la voix (Phase 2) | `id`, `station_id`, `conversation_id`, `caller_phone`, `duration_s`, `outcome`, `failure_reason`, `transcript`, `recording_path`, `created_at` |
| `closures` | Fermetures (Shabbat, fêtes) | `id`, `station_id`, `starts_at`, `ends_at`, `reason` |

Contraintes sur `drivers` : `unique (station_id, phone_e164)`, `unique (station_id, telegram_user_id)`, `unique (auth_user_id)`.

Valeurs de `ride_events` :

- `type` : `created`, `posted`, `claimed`, `assigned`, `claim_rejected`, `done`, `cancelled`, `no_show`.
- `actor_type` : `dispatcher`, `driver`, `voice`, `system`.
- `payload` : selon le type. Pour un claim ou une attribution : `driver_id` et `via` (`telegram_bot`, `miniapp`, `dashboard`). Pour un refus : en plus, `reason`.

## Changements pour la Mini App — et pourquoi

| Changement | Pourquoi |
| --- | --- |
| `drivers.telegram_user_id` en `bigint`, unique par station | Les identifiants Telegram dépassent 32 bits. Un compte Telegram = un chauffeur par station : c'est la clé de l'authentification. |
| `drivers.telegram_linked_at` | Savoir si et quand le chauffeur a lié son compte (contrat Q). Mesure du pilote. |
| `drivers.auth_user_id` | Lier le chauffeur à son utilisateur Supabase Auth : le jeton de la Mini App donne `auth.uid()`, le backend en déduit le chauffeur. Permet de révoquer ses sessions. |
| `drivers.is_available` (vrai par défaut), `drivers.available_until` | Disponibilité simple : qui reçoit les notifications. Vrai par défaut : au lancement, même comportement qu'avant (« tous les inscrits »). La fin optionnelle évite de réveiller un chauffeur qui a oublié de se mettre indisponible. |
| `rides.posted_at` | Ordre et ancienneté dans la liste de la Mini App. Mesure « délai avant claim » (`claimed_at - posted_at`). |
| `rides.updated_at` | Ignorer une réponse plus ancienne que ce qui est déjà affiché (signal + relecture, polling). |
| `rides.notes` | Déjà envoyé par `create-ride`, mais absent de la table ; montré au seul chauffeur attribué. |
| `ride_events.station_id` | Règle d'or 7 (la table n'avait pas la station) ; RLS du dashboard ; routage des événements. |
| `ride_events` : types `assigned` et `claim_rejected`, champ `via` | Audit : qui a pris, par quel canal, qui a été refusé et pourquoi. Mesure de la part des claims par canal. |
| `stations.driver_support_phone_e164` | Bouton « appeler le sadran » de la Mini App (règle d'or 5). |
| `place_aliases.station_id` | Règle d'or 7 : correction d'un oubli de la v0, sans lien direct avec la Mini App. |

## Ce qu'on n'ajoute pas — et pourquoi

| Idée | Pourquoi pas en Phase 1 |
| --- | --- |
| `ride_offers` (une course offerte à un chauffeur précis) | Le claim est ouvert à tous les éligibles ; les destinataires sont calculés au moment de la diffusion et envoyés dans l'événement D. Une table d'offres ne sert qu'aux offres directes ou au dispatch un par un, pas prévus en Phase 1. |
| `driver_sessions` | Les sessions sont celles de Supabase Auth. Une table maison seulement si le prototype impose l'option (b) de D-011. |
| `driver_availability` (table séparée) | Deux colonnes suffisent ; pas besoin d'historique en Phase 1. |
| `driver_devices`, préférences | Telegram gère l'appareil et les notifications. |
| `ride_claim_events` | `ride_events` couvre déjà claims, attributions et refus. |
| `service_areas`, PostGIS, GPS | L'éligibilité de la Phase 1 ne dépend pas du lieu. À rouvrir avec la disponibilité par zone ou par rayon, après G1. |
| `cities` | `places` (ville + quartier) et `place_aliases` existent déjà. |

## Disponibilité effective

Un chauffeur est disponible si `is_available` est vrai et si `available_until` est nul ou dans le futur. Le backend la calcule au moment de construire les destinataires de D. Passer à « pas disponible » remet `available_until` à nul.

## Statuts d'une course

| De | Vers | Déclenché par |
| --- | --- | --- |
| — | `created` | `create-ride` |
| `created` | `posted` | Backend, dès que la course est envoyée au dispatch |
| `posted` | `claimed` | `claim-ride` (bot), `driver-claim-ride` (Mini App) ou `assign-ride` (dashboard) : la même fonction atomique `claim_ride` |
| `claimed` | `done` | `update-ride-status` → écrit le crédit de 7 % et la commission |
| `created`, `posted`, `claimed` | `cancelled` | `update-ride-status` → rien au grand livre |
| `claimed` | `no_show` | `update-ride-status` → rien au grand livre (à valider avec Yossef) |

Toute autre transition est refusée par le backend. Chaque transition écrit une ligne `ride_events` et émet un signal temps réel, dans la même transaction.

## Claim atomique

Le claim est une seule requête conditionnelle : la course passe à `claimed` seulement si elle est encore `posted` et que le chauffeur est actif dans la même station. Si zéro ligne est modifiée, le backend relit l'état pour expliquer le refus. Le bot, la Mini App et le dashboard appellent la même fonction.

```sql
-- Esquisse, pas une migration.
create function private.claim_ride(
  p_ride_id    bigint,
  p_driver_id  uuid,
  p_actor_type text,  -- 'driver' (bot, Mini App) ou 'dispatcher' (assign-ride)
  p_actor_id   uuid,
  p_via        text   -- 'telegram_bot', 'miniapp' ou 'dashboard'
) returns text        -- 'claimed', 'replay', 'already_taken', 'not_claimable', 'driver_blocked', 'not_found'
language plpgsql security definer set search_path = ''
as $$
declare
  v_station text;
  v_status  text;
  v_owner   uuid;
  v_reason  text;
begin
  -- 1. La transition : une seule requête conditionnelle. Deux appels simultanés
  --    se suivent sur le verrou de la ligne ; le second relit status = 'claimed'
  --    et ne modifie rien.
  update public.rides r
     set status = 'claimed', driver_id = p_driver_id,
         claimed_at = now(), updated_at = now()
   where r.id = p_ride_id
     and r.status = 'posted'
     and exists (select 1 from public.drivers d
                  where d.id = p_driver_id
                    and d.station_id = r.station_id
                    and d.status = 'active')
  returning r.station_id into v_station;

  if found then
    insert into public.ride_events (station_id, ride_id, type, actor_type, actor_id, payload)
    values (v_station, p_ride_id,
            case when p_actor_type = 'dispatcher' then 'assigned' else 'claimed' end,
            p_actor_type, p_actor_id,
            jsonb_build_object('driver_id', p_driver_id, 'via', p_via));
    return 'claimed';
  end if;

  -- 2. Zéro ligne : on relit seulement pour expliquer le refus.
  select r.station_id, r.status, r.driver_id
    into v_station, v_status, v_owner
    from public.rides r
   where r.id = p_ride_id;

  if not found or not exists (select 1 from public.drivers d
                               where d.id = p_driver_id and d.station_id = v_station) then
    return 'not_found';  -- course inconnue ou d'une autre station : on ne révèle rien
  end if;

  if v_status = 'claimed' and v_owner = p_driver_id then
    return 'replay';     -- double appui ou réponse perdue : la course est déjà à lui
  end if;

  v_reason := case
    when exists (select 1 from public.drivers d
                  where d.id = p_driver_id and d.status <> 'active') then 'driver_blocked'
    when v_status = 'claimed' then 'already_taken'
    else 'not_claimable'
  end;

  insert into public.ride_events (station_id, ride_id, type, actor_type, actor_id, payload)
  values (v_station, p_ride_id, 'claim_rejected', p_actor_type, p_actor_id,
          jsonb_build_object('driver_id', p_driver_id, 'via', p_via, 'reason', v_reason));
  return v_reason;
end;
$$;

revoke execute on function private.claim_ride from public, anon, authenticated;
```

Tests obligatoires :

- 20 claims parallèles sur la même course → 1 `claimed`, 19 `already_taken` ; dans `ride_events`, 1 ligne `claimed` et 19 `claim_rejected`.
- Un claim par le bot et un par la Mini App au même instant → un seul gagnant.
- Même chauffeur deux fois → `replay`, une seule ligne `claimed`.
- Chauffeur bloqué, chauffeur d'une autre station, course annulée → la bonne raison, course inchangée.
- Annulation et claim simultanés → un seul des deux s'applique.

## Temps réel : la seule politique des chauffeurs

```sql
-- Esquisse. Réception des signaux Broadcast sur les canaux du chauffeur.
create policy "chauffeur : réception de ses canaux"
on realtime.messages for select to authenticated
using (
  exists (
    select 1 from public.drivers d
     where d.auth_user_id = (select auth.uid())
       and d.status = 'active'
       and realtime.topic() in ('rides:' || d.station_id, 'driver:' || d.id)
  )
);
-- Aucune politique d'insertion : seuls les triggers du backend émettent (realtime.send).
```

## Questions ouvertes (Phase 0)

- Source de vérité des soldes de 7 % : le logiciel de Yossef ou cette base ?
- Les prix viennent-ils d'une table fixe ou d'une négociation dans le groupe ?
- Quelles données exactes contient l'export du logiciel ?
- Durée de conservation des enregistrements d'appels (avis de l'avocat).
- Questions liées à la Mini App (table des sadranim, désistement `claimed → posted`, plusieurs courses par chauffeur, clôture par le chauffeur) : voir « Questions ouvertes » dans [`MINIAPP.md`](MINIAPP.md).
