# Base de données

> **Version 0 — brouillon.** Schéma à finaliser en Phase 0, après analyse de l'export du logiciel de Yossef. Responsable : Oren. Mis à jour le 4 octobre 2026 pour la Mini App chauffeur, puis le 7 octobre 2026 après deux relectures : voir les deux sections « Changements ».

Base : Postgres (Supabase). Toute modification passe par une migration dans `backend/supabase/migrations/`.

## Règles

1. **`station_id` dans chaque table métier**, dès le premier jour.
2. **Argent en agorot** (entiers). Jamais de `float`, qui est inexact. (`numeric` serait exact, mais les entiers évitent les décimales dans le JSON et le JavaScript.) Les pourcentages suivent la règle d'arrondi de « Grand livre : règles de calcul ».
3. **Dates en `timestamptz`**, stockées en UTC.
4. **Statuts modifiés uniquement par des fonctions backend** (SQL ou Edge Functions), jamais par un `UPDATE` venu d'un frontend ou d'un bot.
5. **Le grand livre est en ajout seul.** On ne modifie ni ne supprime une ligne : on corrige par une écriture inverse.
6. **RLS activé sur toutes les tables.** Le dashboard ne voit que les données de sa station.
7. **Pas de données personnelles inutiles.** Les enregistrements d'appels vont dans un bucket privé, avec une durée de conservation définie.
8. **Les chauffeurs n'accèdent à aucune table métier.** Ils passent par les fonctions `driver-*`. Leur seule politique RLS porte sur la réception temps réel (`realtime.messages`), par une fonction d'aide (règle 10).
9. **Une politique RLS ne repose jamais sur le seul rôle `authenticated`.** Chauffeurs et sadranim sont tous deux `authenticated` : la politique vérifie l'appartenance (sadran de la station dans `dispatchers`, chauffeur par sa liaison Telegram active dans `driver_links`), par une fonction d'aide (règle 10).
10. **Les fonctions `security definer` vivent dans le schéma `private`, jamais exposé par l'API, et chacune reçoit ses droits explicitement.** Celles qui modifient des données, comme `claim_ride`, ne sont exécutables que par le backend, jamais par `anon` ni `authenticated`. Seules les fonctions d'aide RLS, en lecture seule, qui ne renvoient qu'un oui ou non (ou la station de l'utilisateur), sont exécutables par `authenticated` : une politique s'exécute avec les droits de l'utilisateur, et ne pourrait pas lire `driver_links`, `drivers` ou `dispatchers` autrement. Voir « Droits des fonctions SQL ».
11. **Chaque changement de statut incrémente `rides.version`.** C'est la version, pas l'heure, qui dit quel état d'une course est le plus récent.
12. **Un jeton de chauffeur ne vaut que pour la liaison Telegram active qui l'a obtenu.** Chaque appel protégé retrouve le chauffeur par cette liaison : délier rend l'ancien jeton inutilisable dès l'appel suivant (D-022).

## Tables (v0)

| Table | Rôle | Colonnes principales |
| --- | --- | --- |
| `stations` | Une station cliente (Driverss d'abord) | `id`, `name`, `brand_name`, `timezone`, `driver_support_phone_e164`, `created_at` |
| `dispatchers` | Sadranim et responsables, utilisateurs du dashboard (D-017) | `id`, `station_id`, `auth_user_id`, `display_name`, `role` (`dispatcher`, `admin`), `status` (`active`, `disabled`), `created_at` |
| `customers` | Clients inscrits | `id`, `station_id`, `phone_e164`, `name`, `institution_id`, `consent_at`, `created_at` |
| `institutions` | Synagogues et associations (codes Nedarim Plus) | `id`, `station_id`, `code`, `name` |
| `places` | Villes et quartiers reconnus | `id`, `station_id`, `city`, `area` |
| `place_aliases` | Variantes de noms (prononciation, yiddish, abréviations) | `id`, `station_id`, `place_id`, `alias` |
| `prices` | Table de prix (si validée en Phase 0) | `id`, `station_id`, `from_place_id`, `to_place_id`, `kind`, `price_agorot` |
| `quotes` | Devis donnés par `get-quote`, valables 15 minutes (D-021) | `id`, `station_id`, `from_place_id`, `to_place_id`, `pickup_time`, `kind`, `passengers`, `price_agorot`, `credit_agorot`, `commission_agorot`, `expires_at`, `created_at` |
| `drivers` | Chauffeurs | `id`, `station_id`, `phone_e164`, `display_name`, `status` (`active`, `blocked`), `is_available`, `available_until`, `relink_requires_admin`, `relink_allowed_until`, `relink_same_account_allowed`, `created_at` |
| `driver_links` | Liaisons Telegram des chauffeurs, une ligne par liaison, chacune avec sa propre identité Supabase (D-022) | `id`, `station_id`, `driver_id`, `telegram_user_id` (`bigint`), `auth_user_id`, `status` (`active`, `revoked`), `linked_at`, `revoked_at`, `revoked_reason` (`changed_account`, `compromised`), `revoked_by`, `auth_user_deleted_at` |
| `driver_events` | Historique des chauffeurs : création, modification, blocage, déblocage, liaison, déliaison (avec sa raison), autorisation d'une nouvelle liaison | `id`, `station_id`, `driver_id`, `type`, `actor_type`, `actor_id`, `payload`, `created_at` |
| `rides` | Courses | `id`, `station_id`, `customer_id`, `source`, `quote_id`, `from_place_id`, `to_place_id`, `pickup_address_text`, `pickup_time`, `kind`, `passengers`, `price_agorot`, `credit_agorot`, `commission_agorot`, `institution_id`, `notes`, `status`, `version`, `driver_id`, `posted_at`, `relaunched_at`, `claimed_at`, `updated_at`, `replaces_ride_id`, `idempotency_key`, `conversation_id`, `created_at` |
| `ride_events` | Historique de chaque course (audit) | `id`, `station_id`, `ride_id`, `type`, `actor_type`, `actor_id`, `payload`, `created_at` |
| `notifications` | File des envois du bot : annonces et corrections (D-019). Aucun contenu stocké : il est calculé quand le bot lit l'envoi | `id`, `station_id`, `ride_id`, `ride_version`, `kind` (`ride_posted`, `ride_relaunch`, `ride_assigned`, `message_update`, `ride_cancelled`), `driver_id`, `driver_link_id`, `bot_message_id`, `after_notification_id`, `status` (`pending`, `reserved`, `sent`, `failed`, `skipped`), `reason`, `possibly_sent`, `attempts`, `available_at`, `reserved_until`, `handed_version`, `handed_state`, `resend_at`, `created_at`, `sent_at` |
| `bot_messages` | Messages connus : chaque message Telegram dont le bot a confirmé l'envoi, avec l'état qu'il affiche (D-019) | `id`, `station_id`, `ride_id`, `driver_id`, `driver_link_id`, `notification_id`, `kind` (`ride_posted`, `ride_relaunch`, `ride_assigned`), `telegram_message_id`, `shown_state` (`available`, `yours`, `taken`, `cancelled`), `shown_version`, `check_after`, `created_at`, `updated_at` |
| `dispatcher_alerts` | Signalements à traiter par un sadran, marqués traités par lui (D-019, D-022) | `id`, `station_id`, `ride_id`, `driver_id`, `kind` (`assignment_not_delivered`, `cancellation_not_delivered`, `bot_blocked`, `driver_unlinked`), `created_at`, `resolved_at`, `resolved_by` |
| `ledger_entries` | Grand livre : crédits de 7 %, commissions, dons | `id`, `station_id`, `ride_id`, `entry_type` (`ride_done`, `reversal`, `adjustment`, `opening_balance`), `account` (`customer_credit`, `institution_donation`, `driver_commission`), `account_owner_id`, `amount_agorot`, `reverses_entry_id`, `correction_id`, `reason`, `actor_id`, `created_at` |
| `calls` | Appels traités par la voix (Phase 2) | `id`, `station_id`, `conversation_id`, `caller_phone`, `duration_s`, `outcome`, `failure_reason`, `transcript`, `recording_path`, `created_at` |
| `closures` | Fermetures (Shabbat, fêtes) | `id`, `station_id`, `starts_at`, `ends_at`, `reason` |

Contraintes :

- `drivers` : `unique (station_id, phone_e164)`.
- `driver_links` : une seule liaison active par chauffeur, `unique (driver_id) where status = 'active'` ; une seule par compte Telegram dans une station, `unique (station_id, telegram_user_id) where status = 'active'` ; `unique (auth_user_id)`.
- `dispatchers` : `unique (auth_user_id)`.
- `notifications` : une seule annonce ou un seul avis d'annulation par course, type et chauffeur, `unique (ride_id, kind, driver_id) where kind <> 'message_update'` (le second avis d'annulation est le même envoi, remis une seconde fois) ; une seule correction en cours par message, `unique (bot_message_id) where kind = 'message_update' and status in ('pending', 'reserved')`.
- `bot_messages` : `unique (driver_link_id, telegram_message_id)`.
- `dispatcher_alerts` : un seul signalement ouvert par course, chauffeur et type.
- `ledger_entries` : voir « Grand livre : règles de calcul ».

La file des envois ne stocke aucun contenu : le texte, et pour une attribution les détails de prise en charge, sont lus dans `rides` au moment où le bot lit l'envoi. Aucune donnée client ne reste dans `notifications` ni dans `bot_messages`.

Valeurs de `ride_events` :

- `type` : `created`, `posted`, `relaunched`, `claimed`, `assigned`, `claim_rejected`, `done`, `cancelled`, `no_show`.
- `actor_type` : `dispatcher`, `driver`, `voice`, `system`.
- `payload` : selon le type. Pour un claim ou une attribution : `driver_id` et `via` (`telegram_bot`, `miniapp`, `dashboard`). Pour un refus : en plus, `reason`.

## Changements pour la Mini App — et pourquoi

| Changement | Pourquoi |
| --- | --- |
| `drivers.telegram_user_id` en `bigint`, unique par station | Les identifiants Telegram dépassent 32 bits. Un compte Telegram = un chauffeur par station : c'est la clé de l'authentification. Depuis le 7 octobre : `driver_links.telegram_user_id` (D-022). |
| `drivers.telegram_linked_at` | Savoir si et quand le chauffeur a lié son compte (contrat Q). Mesure du pilote. Depuis le 7 octobre : `driver_links.linked_at`. |
| `drivers.auth_user_id` | Lier le chauffeur à son utilisateur Supabase Auth : le jeton de la Mini App donne `auth.uid()`, le backend en déduit le chauffeur. Permet de révoquer ses sessions. Depuis le 7 octobre : une identité par liaison, `driver_links.auth_user_id` (D-022). |
| `drivers.is_available` (vrai par défaut), `drivers.available_until` | Disponibilité simple : qui reçoit les notifications. Vrai par défaut : au lancement, même comportement qu'avant (« tous les inscrits »). La fin optionnelle évite de réveiller un chauffeur qui a oublié de se mettre indisponible. |
| `rides.posted_at` | Ordre et ancienneté dans la liste de la Mini App. Mesure « délai avant claim » (`claimed_at - posted_at`). |
| `rides.updated_at` | Information pour l'affichage seulement ; l'ordre des changements se juge sur `rides.version` (voir plus bas). |
| `rides.notes` | Déjà envoyé par `create-ride`, mais absent de la table ; montré au seul chauffeur attribué. |
| `ride_events.station_id` | Règle d'or 7 (la table n'avait pas la station) ; RLS du dashboard ; routage des événements. |
| `ride_events` : types `assigned` et `claim_rejected`, champ `via` | Audit : qui a pris, par quel canal, qui a été refusé et pourquoi. Mesure de la part des claims par canal. |
| `stations.driver_support_phone_e164` | Bouton « appeler le sadran » de la Mini App (règle d'or 5). |
| `place_aliases.station_id` | Règle d'or 7 : correction d'un oubli de la v0, sans lien direct avec la Mini App. |

## Changements de la relecture du 7 octobre 2026 — et pourquoi

| Changement | Pourquoi |
| --- | --- |
| `dispatchers` (station, rôle `dispatcher` ou `admin`) | Le dashboard a besoin d'une identité et de droits ; ses politiques RLS en dépendent (D-017). |
| `driver_events` | Journal des actions sur les chauffeurs (création, blocage, déliaison…) : qui, quoi, quand. |
| `quotes`, `rides.quote_id` | Le prix enregistré est celui du devis accepté par le client (D-021). |
| `rides.commission_agorot` | Commission calculée une seule fois, avec la règle d'arrondi (D-018). |
| `rides.version` | Ordre fiable des changements (D-021). `updated_at = now()` ne l'est pas : `now()` donne l'heure du début de la transaction, donc une transaction commencée plus tôt mais validée plus tard reçoit une heure plus ancienne. |
| `rides.relaunched_at`, type `relaunched` | La relance n'a lieu qu'une fois. Elle se pose par une requête conditionnelle qui verrouille la course, comme un claim : un claim simultané passe avant ou après elle, jamais en même temps. Le dashboard montre les courses relancées encore libres (D-019). |
| `rides.replaces_ride_id` | Course recréée après le désistement d'un chauffeur : lien vers l'ancienne, pour que les statistiques comptent un désistement et non une annulation du client. |
| `notifications` | File des envois du bot, en annonces et corrections, avec l'état « peut-être parti » quand Telegram a pu recevoir un message que le bot n'a pas confirmé (D-019). `resend_at` programme le second avis d'annulation qui suit une attribution à l'issue incertaine. |
| `bot_messages` | Chaque message confirmé, relance et message du gagnant compris, peut être corrigé. Un même envoi peut laisser deux messages (confirmation tardive) : les deux sont suivis (D-019). `check_after` programme la vérification d'un message après une modification à l'issue incertaine : un délai dépassé côté bot ne prouve pas que Telegram n'a rien fait. |
| `dispatcher_alerts` | Quand un envoi ne peut pas aboutir (attribution ou avis d'annulation non remis, bot bloqué, chauffeur délié avec des courses en cours), le sadran doit le savoir et appeler le chauffeur. |
| `driver_links`, `drivers.relink_*` | Une liaison Telegram = une identité Supabase ; délier coupe l'accès dès l'appel suivant ; après un vol, la nouvelle liaison passe par un admin (D-022). Remplace `drivers.telegram_user_id`, `telegram_linked_at` et `auth_user_id`. |
| `ledger_entries.entry_type` (dont `adjustment`), `reverses_entry_id`, `correction_id`, `reason`, `actor_id`, index uniques | Pas de double crédit ; corrections complètes, rejouables sans double effet, sans jamais réécrire l'historique ; une seule reprise par compte (D-018). |
| Fonctions d'aide `private.driver_can_listen`, `private.dispatcher_station` | Une politique RLS s'exécute avec les droits de l'utilisateur : elle ne peut pas lire `driver_links`, `drivers` ou `dispatchers`. Ces fonctions vérifient l'appartenance sans ouvrir les tables. |
| Droits explicites de chaque fonction de `private`, contrôle des droits effectifs | `alter default privileges in schema … revoke … from public` n'a aucun effet : PostgreSQL ne retire pas, schéma par schéma, un droit donné à tous par défaut. |

## Ce qu'on n'ajoute pas — et pourquoi

| Idée | Pourquoi pas en Phase 1 |
| --- | --- |
| `ride_offers` (une course offerte à un chauffeur précis) | Le claim est ouvert à tous les éligibles ; les destinataires sont calculés au moment de la diffusion et déposés dans la file des envois (`notifications`). Une table d'offres ne sert qu'aux offres directes ou au dispatch un par un, pas prévus en Phase 1. |
| `driver_sessions` | Les sessions sont celles de Supabase Auth. Une table maison seulement si le prototype impose l'option (b) de D-011. |
| `driver_availability` (table séparée) | Deux colonnes suffisent ; pas besoin d'historique en Phase 1. |
| `driver_devices`, préférences | Telegram gère l'appareil et les notifications. |
| `ride_claim_events` | `ride_events` couvre déjà claims, attributions et refus. |
| `service_areas`, PostGIS, GPS | L'éligibilité de la Phase 1 ne dépend pas du lieu. À rouvrir avec la disponibilité par zone ou par rayon, après G1. |
| `cities` | `places` (ville + quartier) et `place_aliases` existent déjà. |

## Disponibilité effective

Un chauffeur est disponible si `is_available` est vrai et si `available_until` est nul ou dans le futur. Le backend la calcule au moment de créer les annonces « nouvelle course » et celles de la relance. Passer à « pas disponible » remet `available_until` à nul.

## Statuts d'une course

| De | Vers | Déclenché par |
| --- | --- | --- |
| — | `created` | `create-ride` |
| `created` | `posted` | Backend, dès que la course est envoyée au dispatch |
| `posted` | `claimed` | `claim-ride` (bot), `driver-claim-ride` (Mini App) ou `assign-ride` (dashboard) : la même fonction atomique `claim_ride` |
| `claimed` | `done` | `update-ride-status` → écrit le crédit de 7 % et la commission |
| `created`, `posted`, `claimed` | `cancelled` | `update-ride-status` → rien au grand livre |
| `claimed` | `no_show` | `update-ride-status` → rien au grand livre (à valider avec Yossef) |

Toute autre transition est refusée par le backend. Chaque transition fait, dans la même transaction : `version` + 1, une ligne `ride_events`, un signal temps réel, et les envois du bot dans `notifications` : les annonces, et une correction pour chaque message connu de la course qui n'est plus juste (D-019). Avec `expected_version`, `update-ride-status` ne s'applique que si la course est encore à la version que le sadran a vue (D-021).

## Grand livre : règles de calcul

Proposition D-018, à valider avec Yossef.

- **Calcul unique.** La commission (12 %) et le crédit (7 %) sont calculés au devis, en agorot entières, puis verrouillés avec la course (`commission_agorot`, `credit_agorot`). Le passage à `done` les écrit tels quels.
- **Arrondi.** Chaque montant est arrondi une seule fois ; la part technologie est la différence, commission − crédit, donc le total tombe toujours juste. Exemple pour 19,90 ₪ : commission 238,8 → 239 agorot ; crédit 139,3 → 139 ; technologie 100. Le mode d'arrondi est **celui du logiciel de Yossef**, à vérifier sur l'export : sinon, la porte G1 (« soldes concordants ») échoue.
- **Pas de double crédit.** Au passage à `done`, une seule écriture `ride_done` par course et par compte, garantie par un index unique, même si l'opération est rejouée.
- **Corrections.** Jamais de modification ni de suppression. Une correction est faite par un `admin`, dans **une seule transaction** :
  1. s'il existe une écriture fausse, une écriture `reversal` l'annule exactement : montant opposé, même compte, même titulaire, `reverses_entry_id` vers elle ;
  2. si le bon montant n'est pas nul, une écriture `adjustment` porte ce bon montant **en entier**, jamais une différence.

  Une écriture ne peut être inversée qu'une fois, et une écriture `reversal` ne s'inverse pas. Pour corriger une écriture déjà corrigée, on corrige de la même façon la dernière écriture valable, c'est-à-dire l'`adjustment` : l'historique ne change jamais. Chaque correction porte un identifiant fourni par l'appelant (`correction_id`), sa raison et son auteur ; rejouée avec le même identifiant, elle n'écrit rien de plus.
- **Reprise des soldes.** Si notre base devient la source de vérité des soldes (décision de Phase 0), l'import ouvre chaque solde par une écriture `opening_balance`, une seule par station, compte et titulaire, sans recréditer les courses passées. Une reprise fausse se corrige comme toute écriture.
- **Plus tard (Phase 3) :** commission due ou encaissée, utilisation du crédit par le client, versements aux institutions.

Exemple : une course close par erreur a crédité 139 agorot au client.

| Cas | Écritures (une transaction par correction) | Solde |
| --- | --- | --- |
| Rien n'était dû | Correction A : `reversal` −139 | 0 |
| 126 étaient dus | Correction A : `reversal` −139, puis `adjustment` +126 | 126 |
| Puis on découvre que 130 étaient dus | Correction B : `reversal` −126 (inverse l'`adjustment` de A), puis `adjustment` +130 | 130 |

```sql
-- Esquisse, pas une migration : ce que la base garantit.
create unique index ledger_ride_done_once on public.ledger_entries (ride_id, account)
  where entry_type = 'ride_done';                  -- pas de double crédit
create unique index ledger_reversed_once on public.ledger_entries (reverses_entry_id)
  where entry_type = 'reversal';                   -- une seule inverse par écriture
create unique index ledger_correction_once on public.ledger_entries (correction_id, entry_type)
  where entry_type in ('reversal', 'adjustment');  -- correction rejouée : rien de plus
create unique index ledger_opening_once on public.ledger_entries (station_id, account, account_owner_id)
  where entry_type = 'opening_balance';            -- une seule reprise par compte

alter table public.ledger_entries add constraint ledger_correction_fields check (
  entry_type not in ('reversal', 'adjustment')
  or (correction_id is not null and reason is not null and actor_id is not null)
);
alter table public.ledger_entries add constraint ledger_reversal_target check (
  (entry_type = 'reversal') = (reverses_entry_id is not null)
);
-- La fonction de correction vérifie le reste, dans la même transaction : l'inverse porte
-- le montant opposé, le même compte et le même titulaire ; une écriture `reversal` ne
-- s'inverse pas.
```

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
         claimed_at = now(), updated_at = now(), version = r.version + 1
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
    -- Dans la même transaction : signal temps réel, annonce de l'attribution au gagnant
    -- et correction de chaque message connu de la course (D-019).
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
-- Appelée seulement par les Edge Functions du backend, par une connexion directe à
-- Postgres : le schéma `private` n'est pas exposé par l'API.
```

Tests obligatoires :

- 20 claims parallèles sur la même course → 1 `claimed`, 19 `already_taken` ; dans `ride_events`, 1 ligne `claimed` et 19 `claim_rejected`.
- Un claim par le bot et un par la Mini App au même instant → un seul gagnant.
- Même chauffeur deux fois → `replay`, une seule ligne `claimed`.
- Chauffeur bloqué, chauffeur d'une autre station, course annulée → la bonne raison, course inchangée.
- Annulation et claim au même instant : annulation d'abord → claim refusé (`not_claimable`) ; claim d'abord → la course est prise puis annulée, et le chauffeur est prévenu ; annulation avec une `expected_version` périmée → `version_conflict`, course inchangée.
- Relance et claim au même instant : claim d'abord → aucune relance ; relance d'abord → le claim attend son verrou, puis chaque message de relance confirmé est corrigé. Jamais deux relances pour une course.

## Temps réel : la seule politique des chauffeurs

```sql
-- Esquisse. Les chauffeurs ne peuvent pas lire `driver_links` ni `drivers` : la politique
-- passe par une fonction d'aide qui ne répond qu'à « le chauffeur connecté, par sa liaison
-- Telegram active, peut-il écouter ce canal ? ».
create function private.driver_can_listen(p_topic text)
returns boolean
language sql stable security definer set search_path = ''
as $$
  select exists (
    select 1
      from public.driver_links l
      join public.drivers d on d.id = l.driver_id
     where l.auth_user_id = (select auth.uid())
       and l.status = 'active'
       and d.status = 'active'
       and p_topic in ('rides:' || d.station_id, 'driver:' || d.id)
  );
$$;

-- Droits explicites, juste après la création (voir « Droits des fonctions SQL »).
revoke execute on function private.driver_can_listen(text) from public, anon, authenticated;
grant usage on schema private to authenticated;
grant execute on function private.driver_can_listen(text) to authenticated;

create policy "chauffeur : réception de ses canaux"
on realtime.messages for select to authenticated
using (
  realtime.messages.extension = 'broadcast'
  and private.driver_can_listen(realtime.topic())
);
-- Aucune politique d'insertion : seuls les triggers du backend émettent (realtime.send).
```

Les politiques du dashboard suivent le même principe : `private.dispatcher_station()` renvoie la station du sadran connecté (ou rien), et chaque politique compare `station_id` à cette valeur.

Supabase Realtime vérifie la politique quand le chauffeur rejoint un canal, puis garde le résultat tant que la connexion vit ou jusqu'à un nouveau jeton. Un canal déjà ouvert peut donc encore recevoir des signaux après un blocage ou une déliaison, jusqu'à l'expiration du jeton (une heure au plus) : c'est pourquoi les signaux ne portent jamais de donnée personnelle (contrat P).

Test obligatoire : un chauffeur actif peut s'abonner à ses deux canaux, mais pas à ceux d'un autre chauffeur ni d'une autre station ; un chauffeur bloqué, ou dont la liaison est révoquée, ne peut plus s'abonner.

## Droits des fonctions SQL

Dans PostgreSQL, une fonction est exécutable par tous (`PUBLIC`) dès sa création. `alter default privileges in schema private revoke execute on functions from public` n'y change rien : un droit donné à tous par défaut ne se retire pas schéma par schéma. Chaque migration qui crée une fonction dans `private` écrit donc ses droits elle-même : `revoke execute … from public, anon, authenticated`, puis seulement les `grant` de ce tableau.

| Objet | `anon` | `authenticated` | Utilisé par |
| --- | --- | --- | --- |
| `private.claim_ride` | Non | Non | Edge Functions du backend, par une connexion directe |
| Autres fonctions de `private` qui écrivent (relance, file des envois, liaisons, corrections du grand livre) | Non | Non | Backend |
| `private.driver_can_listen(text)` | Non | Oui | Politique de `realtime.messages` |
| `private.dispatcher_station()` | Non | Oui | Politiques RLS du dashboard |
| Schéma `private` (`usage`) | Non | Oui, pour les deux fonctions d'aide | — |

Contrôle des droits effectifs, lancé par les tests après chaque migration :

```sql
-- 1. Fonctions de `private` qu'anon ou authenticated peuvent exécuter, hors des deux
--    fonctions d'aide. Résultat attendu : aucune ligne.
select r.role, p.oid::regprocedure as fonction
  from pg_proc p
  join pg_namespace n on n.oid = p.pronamespace
 cross join (values ('anon'), ('authenticated')) as r(role)
 where n.nspname = 'private'
   and has_function_privilege(r.role, p.oid, 'execute')
   and not (r.role = 'authenticated'
            and p.oid in ('private.driver_can_listen(text)'::regprocedure,
                          'private.dispatcher_station()'::regprocedure));

-- 2. anon n'a pas accès au schéma. Résultat attendu : false.
select has_schema_privilege('anon', 'private', 'usage');
```

## Questions ouvertes (Phase 0)

- Source de vérité des soldes de 7 % : le logiciel de Yossef ou cette base ? De la réponse dépend la reprise des soldes (`opening_balance`) ou la synchro.
- Les prix viennent-ils d'une table fixe ou d'une négociation dans le groupe ?
- Quelles données exactes contient l'export du logiciel ? Quel mode d'arrondi applique-t-il (D-018) ?
- Durée de conservation des enregistrements d'appels (avis de l'avocat).
- Commission due ou encaissée, utilisation du crédit, versements aux institutions : à décrire avant la Phase 3.
- Une course close par erreur (`done` au lieu de `no_show`) : corriger aussi son statut, ou seulement le grand livre (D-018) ? À trancher avec Yossef avant le pilote.
- Questions liées à la Mini App (désistement `claimed → posted`, plusieurs courses par chauffeur, clôture par le chauffeur) : voir « Questions ouvertes » dans [`MINIAPP.md`](MINIAPP.md).
