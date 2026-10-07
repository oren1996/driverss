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
    │   ├── manage-driver/           ← dashboard (rôle admin) : créer, bloquer, délier, autoriser une nouvelle liaison
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
- L'initData Telegram n'est validé que dans `driver-auth-telegram` (signature, `auth_date` de moins de 300 s). Ensuite, le chauffeur vient du jeton, par sa liaison Telegram **active** (`driver_links.auth_user_id`) ; liaison et statut sont relus à chaque appel. Jamais d'identité prise dans le corps de la requête.
- Liaisons (D-022) : une identité Supabase par liaison. Délier passe d'abord la liaison à `revoked` en base, dans la même transaction que l'abandon des envois en attente vers elle et le signalement des courses en cours du chauffeur ; ensuite seulement, l'identité Supabase est supprimée, en réessayant jusqu'à réussir. Après une déliaison `compromised`, Q refuse toute liaison hors de la fenêtre ouverte par `allow_relink`, et l'ancien compte Telegram sauf accord explicite de l'admin.
- Les changements de statut passent par des fonctions SQL atomiques. Le claim et l'attribution passent par une seule fonction, `claim_ride` (esquisse dans `docs/DATABASE.md`) : une requête conditionnelle (`posted` → `claimed`) ; zéro ligne modifiée = refus expliqué (`already_taken`, `replay` pour le même chauffeur…). Elle vit dans le schéma `private`, jamais exposé par l'API, et n'est pas exécutable par `anon` ni `authenticated`.
- Chaque changement de statut fait, dans la même transaction : `version` + 1, une ligne `ride_events`, un signal Broadcast sans donnée personnelle, et les envois du bot dans `notifications` (D-019). `update-ride-status` respecte `expected_version` quand il est fourni.
- Envois du bot (D-019, contrat T) :
  - deux familles. Une annonce périmée (version changée, liaison révoquée ; pour une diffusion ou une relance, aussi chauffeur bloqué ou heure de prise en charge passée) n'est jamais remise. Une correction ne l'est jamais pour un changement de version : son contenu est calculé à la lecture, selon l'état actuel ;
  - chaque message confirmé devient un message connu (`bot_messages`). À chaque changement de statut et à chaque confirmation, comparer l'état affiché à l'état actuel, et créer une correction s'ils diffèrent. Une seule correction en cours par message ; l'état enregistré d'un message ne recule jamais ;
  - une absence de confirmation ne prouve pas une absence d'envoi : `unknown` ou réservation expirée → « peut-être parti », pour toujours. Une attribution peut-être partie, puis annulée, déclenche l'avis d'annulation. Un envoi qui ne peut pas aboutir crée le signalement prévu (`dispatcher_alerts`) ;
  - la file ne stocke aucun contenu, donc aucune donnée client.
- La relance à 60 s est une tâche planifiée qui verrouille la course comme une transition (requête conditionnelle : encore `posted`, pas encore relancée), puis crée ses annonces dans la même transaction, avec des destinataires recalculés. Pendant une fermeture (`closures`), rien ne part et aucune relance n'a lieu ; tout est réévalué à la réouverture (D-020).
- `create-ride` n'accepte qu'un devis valide (`quote_id`, D-021) : le prix enregistré est celui du devis.
- Le grand livre (`ledger_entries`) est en ajout seul. Le crédit de 7 % et la commission sont écrits au passage à `done`, dans la même transaction, avec la règle d'arrondi de D-018 et une seule écriture par course et par compte. Une correction (D-018, `admin` seulement) se fait dans une transaction : l'inverse de l'écriture fausse, puis, si besoin, un `adjustment` du bon montant en entier ; une seule inverse par écriture ; le `correction_id` la rend rejouable sans double effet.
- L'éligibilité se calcule ici seulement : annonces D et relance, liste `driver-rides`, claim.
- Les chauffeurs n'ont aucune politique RLS sur les tables métier. Leur seule politique : la réception de leurs canaux sur `realtime.messages`, par la fonction d'aide `private.driver_can_listen`, qui vérifie la liaison active.
- Droits SQL : chaque fonction de `private` reçoit ses droits dans sa migration (`revoke execute … from public, anon, authenticated`, puis seulement les `grant` prévus). Jamais `alter default privileges in schema … revoke … from public`, qui n'a aucun effet. Les tests contrôlent les droits effectifs (requête dans `docs/DATABASE.md`).
- La clé `service_role` ne sort jamais du serveur.
- MCP Supabase (plugin `supabase`) : projet de développement uniquement, jamais la production ; lecture seule, limité à ce projet. Ne jamais l'utiliser pour modifier le schéma ou les données : tout changement passe par une migration dans `backend/supabase/migrations/`.
- `seed.sql` ne contient que des données inventées. Jamais de vraies données de Yossef.
- Logs sans secrets, sans initData, sans jetons ni numéros de téléphone complets.

## Tests obligatoires avant fusion

Ce sont des tests fonctionnels : ils seront écrits et exécutés pendant l'implémentation (Phase 1). Aujourd'hui, seule la documentation existe.

- Tests unitaires et d'API de chaque endpoint.
- Mauvais secret → refus.
- Tableau des droits : chaque fonction refuse les appelants qui ne sont pas sur sa ligne ; un sadran d'une autre station → refus ; `manage-driver` sans le rôle `admin` → refus.
- Droits SQL effectifs : après chaque migration, la requête de contrôle de `docs/DATABASE.md` ne renvoie aucune ligne.
- Ville inconnue → `unknown_place`.
- Devis expiré → `quote_expired` ; le prix enregistré est celui du devis, même si la table de prix a changé entre-temps.
- Deux claims simultanés → un seul gagnant, y compris un par le bot et un par la Mini App ; 20 claims parallèles → 1 `claimed`, 19 `already_taken`, 20 lignes dans `ride_events`.
- Même chauffeur deux fois → `replay`, une seule ligne `claimed`.
- Claim puis annulation : la course est prise puis annulée, et le chauffeur est prévenu ; annulation avec une `expected_version` périmée → `version_conflict`, course inchangée.
- Même `idempotency_key` deux fois → une seule course.
- Passage à `done` rejoué → une seule écriture par compte au grand livre.
- File des envois : une attribution jamais lue par le bot, puis la course annulée → attribution abandonnée, sans avis d'annulation ; un envoi non confirmé revient après 60 s ; un chauffeur qui a reçu la diffusion et la relance voit ses deux messages corrigés, gagnant compris, quel que soit le canal du claim.
- initData falsifié, trop vieux ou d'un autre bot → refus.
- Jeton de chauffeur : lecture directe de `rides` ou `customers` → zéro ligne ; appel RPC de `claim_ride` → refus.
- Temps réel : un chauffeur actif peut s'abonner à ses deux canaux, pas à ceux d'un autre chauffeur ni d'une autre station ; un chauffeur bloqué ou délié ne peut plus s'abonner.
- Chauffeur bloqué → refus immédiat sur toutes les fonctions `driver-*`.
- Liste `available`, signaux temps réel, annonces D et messages modifiés : aucun champ client.

Scénarios de validation des envois, des liaisons et du grand livre :

1. **Envoi sans confirmation suivi d'une annulation.** Le bot lit l'attribution, Telegram la reçoit, le bot s'arrête sans confirmer ; le sadran annule la course. Attendu : l'attribution n'est pas renvoyée ; à la fin de sa réservation, elle est « peut-être partie » et l'avis d'annulation est remis au bot ; si l'avis ne peut pas partir, un signalement `cancellation_not_delivered` apparaît.
2. **Confirmation tardive.** Une diffusion est confirmée `sent` après la fin de sa réservation, alors qu'un autre chauffeur a pris la course entre-temps. Attendu : le message est enregistré comme message connu, puis une correction `taken` est créée et part.
3. **Corrections reçues dans le désordre.** La course est prise puis annulée pendant qu'une correction du même message est en cours ; la confirmation d'une tentative dépassée arrive après celle d'une tentative plus récente. Attendu : jamais deux corrections en cours pour le même message ; l'état enregistré ne recule pas ; le message finit sur `cancelled`.
4. **Relance et claim simultanés.** Relance et claim lancés au même instant, de nombreuses fois. Attendu : jamais de relance après un claim ; chaque message de relance confirmé après le claim est corrigé ; jamais deux relances pour une course.
5. **Déliaison puis nouvelle liaison.** Connexion (J), déliaison (S), nouvelle liaison (Q), nouvelle connexion. Attendu : l'ancien jeton est refusé par toutes les fonctions `driver-*` et pour un nouvel abonnement temps réel, même avant son expiration. Si la suppression de l'identité Supabase échoue (panne simulée), un jeton rafraîchi reste refusé ; une fois l'identité supprimée, le rafraîchissement lui-même est refusé. Le nouveau jeton marche ; `driver_id`, courses et grand livre sont inchangés ; les envois en attente vers l'ancienne liaison sont abandonnés.
6. **Tentative de réassociation de l'ancien compte.** Après une déliaison `compromised`, Q depuis l'ancien compte → `link_requires_admin`. Après `allow_relink` sans `allow_same_account` : l'ancien compte est encore refusé ; un nouveau compte est accepté pendant 15 minutes, puis refusé. Avec `allow_same_account` : l'ancien compte est accepté pendant la fenêtre.
7. **Fermeture Shabbat.** Pendant une fermeture, une course prise est annulée et des annonces attendent. Attendu : `pull` ne renvoie rien et aucune relance n'a lieu ; à la réouverture, les corrections partent avec l'état du moment ; les annonces périmées, et les diffusions dont l'heure de prise en charge est passée, sont abandonnées.
8. **Correction comptable rejouée.** La même correction (même `correction_id`) est envoyée deux fois. Attendu : une seule `reversal` et au plus un `adjustment` ; inverser une deuxième fois la même écriture → refus ; corriger ensuite l'`adjustment` → accepté ; solde final juste ; aucune ligne modifiée ni supprimée.
