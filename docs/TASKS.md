# Tâches

Le détail et le contexte de chaque tâche sont dans [`PLAN.md`](PLAN.md). Ici, on suit l'avancement. Format : `- [ ] tâche — responsable`.

Les tâches de la Mini App chauffeur portent un identifiant `TMA-xx` parce qu'elles dépendent les unes des autres : « après TMA-xx » donne l'ordre. Spec : [`MINIAPP.md`](MINIAPP.md).

## Maintenant — Phase 0 (semaines 1 à 3)

- [ ] Présenter le plan à Yossef, réunion de lancement à cinq — Papa
- [ ] Obtenir l'export du logiciel, 20 enregistrements d'appels, 50 messages de courses, la liste des chauffeurs actifs — Papa
- [ ] Repérer 10 chauffeurs testeurs et noter leur téléphone — Papa
- [ ] Annoncer le passage à Telegram et aider à l'installer — Papa
- [ ] Négocier et faire rédiger l'accord avec Yossef — Eitan
- [ ] Pacte écrit entre nous quatre (qui décide, comment on partage) — Eitan
- [ ] Trouver un avocat et lui envoyer les questions — Eitan
- [ ] Créer le compte ElevenLabs, démo sur les 20 enregistrements — Eitan
- [ ] Analyser l'export (panier moyen, interurbain, clients actifs, courses par chauffeur, recouvrement) — Oren
- [ ] Décider la source de vérité des soldes de 7 % — Oren
- [ ] Décider le modèle de prix, avec Papa — Oren
- [ ] Finaliser le schéma v0 (`DATABASE.md`) et les contrats v0 (`API_CONTRACTS.md`) — Oren
- [ ] Vérifier que l'exclusivité ne bloque pas le projet de colis d'Oren — Oren
- [ ] Observer un sadran sur place et écrire le déroulé — Ilan
- [ ] Collecter les codes et formats de messages des chauffeurs — Ilan
- [ ] Valider Telegram une semaine avec les 10 chauffeurs, lister ceux qui ne peuvent pas l'installer — Ilan
- [ ] Prise en main de Supabase, deux sessions en binôme avec Oren — Ilan
- [ ] Installer les plugins Claude Code recommandés, chacun ceux de son rôle (tableau dans `README.md`) — Tous
- [ ] Remplir les heures disponibles de chacun dans `PLAN.md` — Tous
- [ ] **Porte G0** : avis juridique acceptable, accord signé, données reçues — Tous

### Mini App chauffeur — Phase 0

- [ ] TMA-01 · Relire la spec v0 (`MINIAPP.md`, contrats J à R, décisions D-009 à D-016) : valider, corriger ou refuser — Ilan, Oren
- [ ] TMA-02 · Pendant le test Telegram, ouvrir une Mini App de démonstration jetable sur les téléphones des 10 chauffeurs : filtres, version de Telegram, temps réel — Ilan, avec Papa
- [ ] TMA-03 · Prototype jetable dans une branche : initData → session Supabase → canal Realtime privé ; trancher l'option a ou b de D-011 — Oren

## Ensuite — Phase 1 (semaines 4 à 9)

- [ ] Schéma Supabase, RLS, `station_id` partout — Oren
- [ ] Import des données de Yossef — Oren
- [ ] Endpoints : `create-ride`, `get-quote`, `claim-ride`, `assign-ride`, `update-ride-status`, `GET /rides/{id}` — Oren
- [ ] Grand livre : 7 % et commission au statut `done` — Oren
- [ ] Synchro avec le logiciel de Yossef — Oren
- [ ] Sécurité : secrets, signatures, pas de numéro client dans les messages de groupe — Oren
- [ ] Dashboard du sadran (mobile, hébreu, droite à gauche) — Ilan
- [ ] Bot Telegram : inscription, diffusion, bouton « je prends », message privé — Ilan
- [ ] Repli : relance à 60 s puis attribution manuelle — Ilan
- [ ] Pilote 10 chauffeurs, puis tous ; mesures hebdomadaires — Ilan
- [ ] Prototype vocal hors production, dictionnaire des lieux — Eitan
- [ ] Suivi Nedarim Plus (date, API, affiches) — Eitan
- [ ] Formation des sadranim et des chauffeurs — Papa
- [ ] **Porte G1** : 2 semaines, 100 % des courses dans le système, 0 double attribution, soldes concordants — Tous

La porte G1 ne dépend pas de la Mini App : le bouton du bot reste le chemin garanti (D-009).

### Mini App chauffeur — Phase 1, dans l'ordre

- [ ] TMA-04 · Schéma : colonnes des chauffeurs, `rides.posted_at` et `updated_at`, `ride_events.station_id` ; fonction `claim_ride` et ses tests de concurrence — Oren
- [ ] TMA-05 · Gestion des chauffeurs : contrat de pré-inscription, blocage avec révocation des sessions et déliaison (à écrire) ; `link-driver-telegram` (Q) ; inscription par partage du contact dans le bot — Oren, Ilan (après TMA-04)
- [ ] TMA-06 · `driver-auth-telegram` (J) et `driver-me` (K) — Oren (après TMA-03, TMA-05)
- [ ] TMA-07 · Squelette de la Mini App contre les mocks : session, profil, états, hébreu de droite à gauche — Ilan (en parallèle de TMA-04 à TMA-06)
- [ ] TMA-08 · Courses disponibles et détail : `driver-rides` (M, N) côté backend ; écrans de la Mini App — Oren, Ilan (après TMA-06, TMA-07)
- [ ] TMA-09 · Claim : `driver-claim-ride` (O), `claim-ride` (E) aligné, événement `ride_status_changed` (R) ; bot : message privé au gagnant et « נלקחה » aux autres ; Mini App : écrans gagnant et perdant — Oren, Ilan (après TMA-08)
- [ ] TMA-10 · Temps réel : trigger et signaux (P), politique sur `realtime.messages` ; abonnement et repli par polling dans la Mini App — Oren, Ilan (après TMA-09)
- [ ] TMA-11 · Disponibilité : `driver-set-availability` (L), `recipients` dans D, interrupteur — Oren, Ilan (après TMA-06)
- [ ] TMA-12 · Test de bout en bout : course créée au dashboard, notifiée par le bot, deux claims simultanés (bot et Mini App) ; un seul gagnant, `rides` et `ride_events` corrects en base — Ilan, Oren (après TMA-09 à TMA-11)
- [ ] TMA-13 · Pilote : Mini App ouverte aux 10 chauffeurs, puis à tous ; mesures hebdomadaires (claims par canal, échecs d'ouverture) — Ilan, Papa (après TMA-12)

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
- [x] Spec v0 de la Mini App chauffeur : architecture, contrats J à R, schéma, décisions D-009 à D-016 (proposées) — Oren
