# Tâches

Le détail et le contexte de chaque tâche sont dans [`PLAN.md`](PLAN.md). Ici, on suit l'avancement. Format : `- [ ] tâche — responsable`.

Deux pistes portent un identifiant, parce que leurs tâches dépendent les unes des autres : « après XXX-nn » donne l'ordre.

- `SOC-xx` : le **socle** — backend, bot et dashboard. C'est le chemin de la porte G1.
- `TMA-xx` : la **Mini App** chauffeur ([`MINIAPP.md`](MINIAPP.md)). Elle dépend du socle, jamais l'inverse.

## Maintenant — Phase 0 (semaines 1 à 3)

- [ ] Passer le dépôt GitHub en privé ; décider s'il faut GitHub Pro pour protéger `main` — Oren
- [ ] Ajouter Eitan et Ilan comme collaborateurs du dépôt GitHub (indispensable une fois le dépôt privé, et pour qu'Ilan puisse être désigné relecteur), puis renseigner leurs noms dans `.github/CODEOWNERS` — Oren
- [ ] Présenter le plan à Yossef, réunion de lancement à cinq — Papa
- [ ] Obtenir l'export du logiciel, 20 enregistrements d'appels, 50 messages de courses, la liste des chauffeurs actifs — Papa
- [ ] Repérer 10 chauffeurs testeurs et noter leur téléphone — Papa
- [ ] Annoncer le passage à Telegram et aider à l'installer — Papa
- [ ] Négocier et faire rédiger l'accord avec Yossef — Eitan
- [ ] Pacte écrit entre nous quatre (qui décide, comment on partage) — Eitan
- [ ] Trouver un avocat et lui envoyer les questions — Eitan
- [ ] Créer le compte ElevenLabs, démo sur les 20 enregistrements — Eitan
- [ ] Analyser l'export (panier moyen, interurbain, clients actifs, courses par chauffeur, recouvrement), dont la règle d'arrondi du logiciel (D-018) — Oren
- [ ] Décider la source de vérité des soldes de 7 % — Oren
- [ ] Décider le modèle de prix, avec Papa — Oren
- [ ] Finaliser le schéma v0 (`DATABASE.md`) et les contrats v0 (`API_CONTRACTS.md`) — Oren
- [ ] Trancher les décisions proposées D-017 à D-022 (relectures du 7 octobre) — Oren, Ilan ; avec Papa (D-020) et Yossef (D-018) ; Eitan à prévenir pour le devis (D-021)
- [ ] Vérifier que l'exclusivité ne bloque pas le projet de colis d'Oren — Oren
- [ ] Observer un sadran sur place et écrire le déroulé — Ilan
- [ ] Collecter les codes et formats de messages des chauffeurs — Ilan
- [ ] Valider Telegram une semaine avec les 10 chauffeurs, lister ceux qui ne peuvent pas l'installer — Ilan
- [ ] Prise en main de Supabase, deux sessions en binôme avec Oren — Ilan
- [ ] Installer les plugins Claude Code recommandés, chacun ceux de son rôle (tableau dans `README.md`) — Tous
- [ ] Remplir les heures disponibles de chacun dans `PLAN.md` — Tous
- [ ] **Porte G0** : avis juridique acceptable, accord signé, données reçues — Tous

### Mini App chauffeur — Phase 0

- [x] TMA-01 · Relire la spec v0 (`MINIAPP.md`, contrats J à R, décisions D-009 à D-016) : validée par Ilan (PR #1) — Ilan, Oren
- [ ] TMA-02 · Pendant le test Telegram, ouvrir une Mini App de démonstration jetable sur les téléphones des 10 chauffeurs : filtres, version de Telegram, temps réel — Ilan, avec Papa
- [ ] TMA-03 · Prototype jetable dans une branche : initData → session Supabase → canal Realtime privé ; trancher l'option a ou b de D-011 — Oren

## Ensuite — Phase 1 (semaines 4 à 9)

### Socle — dans l'ordre (le chemin de la porte G1)

- [ ] SOC-01 · Schéma : tables (dont `dispatchers`, `quotes`, `driver_links`, `notifications`, `bot_messages`, `dispatcher_alerts`, `driver_events`), RLS par station avec fonctions d'aide, droits explicites de chaque fonction et contrôle des droits effectifs, `station_id` partout, `rides.version` ; fonction `claim_ride` et ses tests de concurrence — Oren
- [ ] SOC-02 · Import des données de Yossef : lieux, prix, clients, chauffeurs pré-inscrits, soldes selon la décision de Phase 0 — Oren (après SOC-01)
- [ ] SOC-03 · Authentification et droits : fonction commune (modules, sadranim, chauffeurs), tableau « Qui peut appeler quoi » appliqué et testé — Oren (après SOC-01)
- [ ] SOC-04 · Devis et création : `get-quote` (B) avec `quote_id`, `create-ride` (C) — Oren (après SOC-03)
- [ ] SOC-05 · Claim et statuts : `claim-ride` (E), `assign-ride` (F), `update-ride-status` (G) avec `expected_version`, `GET /rides/{id}` (H) — Oren (après SOC-03)
- [ ] SOC-06 · Chauffeurs et liaisons Telegram : `manage-driver` (S, dont déliaison avec sa raison et `allow_relink`) et `link-driver-telegram` (Q) ; une identité Supabase par liaison ; procédure « téléphone perdu ou volé » (D-022) — Oren (après SOC-03)
- [ ] SOC-07 · File des envois : annonces et corrections créées avec chaque transition, messages connus, envois peut-être partis et confirmations tardives, signalements, réveil du bot, `bot-outbox` (T), relance planifiée à 60 s qui verrouille la course comme une transition — Oren (après SOC-05, SOC-06)
- [ ] SOC-08 · Bot : inscription par partage du contact (Q), annonces depuis la file au format habituel, corrections des messages et avis d'annulation, confirmations `sent` / `retry` / `failed` / `unknown`, limites de Telegram, bouton « אני לוקח » (E) et correction au clic en secours — Ilan (après SOC-06, SOC-07)
- [ ] SOC-09 · Dashboard du sadran (mobile, hébreu, droite à gauche) : connexion, saisie (B, C), suivi en direct, attribution (F), clôture (G), courses relancées encore libres en évidence, signalements à traiter, gestion des chauffeurs et des liaisons (S) — Ilan (après SOC-04 à SOC-06 ; signalements après SOC-07)
- [ ] SOC-10 · Grand livre : écritures au passage à `done`, règle d'arrondi (D-018), une seule écriture par course et par compte ; corrections par un `admin` (inverse et `adjustment`, rejouables sans double effet), avec leur contrat et leur écran à écrire — Oren ; écran : Ilan (après SOC-05, SOC-09 ; D-018 tranchée)
- [ ] SOC-11 · Fermetures : aucun envoi ni relance pendant Shabbat et les fêtes ; corrections gardées et envoyées à la réouverture (D-020) — Oren, Ilan (après SOC-07)
- [ ] SOC-12 · Procédure écrite de retour au manuel si le backend ou le bot tombe ; essai à blanc avec un sadran — Ilan, Papa
- [ ] SOC-13 · Test de bout en bout du parcours : saisie → devis → diffusion → deux claims simultanés → clôture → solde, plus les scénarios de validation des envois, des liaisons et du grand livre (`backend/AGENTS.md`, `dispatch/AGENTS.md`) ; données finales correctes en base — Ilan, Oren (après SOC-08 à SOC-11)

### Autres tâches de la Phase 1

- [ ] Synchro avec le logiciel de Yossef, selon la décision de Phase 0 — Oren
- [ ] Sécurité : secrets, signatures, pas de numéro client dans un message vu par plusieurs chauffeurs — Oren
- [ ] Pilote 10 chauffeurs, puis tous ; mesures hebdomadaires — Ilan
- [ ] Prototype vocal hors production, dictionnaire des lieux — Eitan
- [ ] Suivi Nedarim Plus (date, API, affiches) — Eitan
- [ ] Formation des sadranim et des chauffeurs — Papa
- [ ] **Porte G1** : 2 semaines, 100 % des courses dans le système, 0 double attribution, soldes concordants — Tous

La porte G1 dépend du socle (SOC-01 à SOC-13), pas de la Mini App : le bouton du bot reste le chemin garanti (D-009).

### Mini App chauffeur — Phase 1, après le socle

- [ ] TMA-04 · `driver-auth-telegram` (J) et `driver-me` (K), liaison active vérifiée à chaque appel (D-022) — Oren (après TMA-03, SOC-03, SOC-06)
- [ ] TMA-05 · Squelette de la Mini App contre les mocks : session, profil, états, hébreu de droite à gauche — Ilan (après SOC-08 : le bot d'abord)
- [ ] TMA-06 · Courses disponibles et détail : `driver-rides` (M, N) côté backend ; écrans de la Mini App — Oren, Ilan (après TMA-04, TMA-05)
- [ ] TMA-07 · Claim depuis la Mini App : `driver-claim-ride` (O) ; écrans gagnant et perdant — Oren, Ilan (après TMA-06, SOC-05)
- [ ] TMA-08 · Temps réel : signaux (P), fonction d'aide et politique sur `realtime.messages` ; abonnement et repli par polling dans la Mini App — Oren, Ilan (après TMA-07)
- [ ] TMA-09 · Disponibilité : `driver-set-availability` (L), interrupteur — Oren, Ilan (après TMA-04)
- [ ] TMA-10 · Test de bout en bout : bot et Mini App ensemble, deux claims simultanés (un par canal) ; un seul gagnant, données correctes en base — Ilan, Oren (après TMA-07 à TMA-09, SOC-13)
- [ ] TMA-11 · Pilote : Mini App ouverte aux 10 chauffeurs, puis à tous ; mesures hebdomadaires (claims par canal, échecs d'ouverture) — Ilan, Papa (après TMA-10)

## Plus tard

- [ ] Phase 2 — agent vocal en production (nuit, débordement, puis tout) — Eitan, Oren, Ilan
- [ ] Phase 3 — facturation des chauffeurs, avis, parrainage avec consentement — Oren, Ilan, Eitan
- [ ] Phase 4 — station pilote sous marque neutre — Eitan, Papa, Oren, Ilan
- [ ] Mini App après G1, selon les mesures du pilote : zones ou rayon (PostGIS), carte, filtres, historique, offres directes — Ilan, Oren

## Fait

- [x] Lecture des trois plans (Yossef, synthèse familiale, répartition) — Oren
- [x] Plan détaillé par phase — Oren
- [x] Décision : Telegram uniquement — Oren
- [x] Création du dépôt et de la documentation v0 — Oren
- [x] Spec v0 de la Mini App chauffeur : architecture, contrats J à R, schéma, décisions D-009 à D-016 — Oren
- [x] Relecture critique de la documentation et corrections : droits des appelants, devis, envois du bot, grand livre, concurrence, planning (D-017 à D-021 proposées) — Oren
- [x] Deuxième relecture et corrections : annonces et corrections des messages du bot, envois peut-être partis, liaisons Telegram et procédure après un vol, corrections du grand livre, droits SQL, `version` dans toutes les vues (D-022 proposée ; D-018 à D-020 précisées) — Oren
