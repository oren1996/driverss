# Tâches

Le détail et le contexte de chaque tâche sont dans [`PLAN.md`](PLAN.md). Ici, on suit l'avancement. Format : `- [ ] tâche — responsable`.

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
- [ ] Remplir les heures disponibles de chacun dans `PLAN.md` — Tous
- [ ] **Porte G0** : avis juridique acceptable, accord signé, données reçues — Tous

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

## Plus tard

- [ ] Phase 2 — agent vocal en production (nuit, débordement, puis tout) — Eitan, Oren, Ilan
- [ ] Phase 3 — facturation des chauffeurs, avis, parrainage avec consentement — Oren, Ilan, Eitan
- [ ] Phase 4 — station pilote sous marque neutre — Eitan, Papa, Oren, Ilan

## Fait

- [x] Lecture des trois plans (Yossef, synthèse familiale, répartition) — Oren
- [x] Plan détaillé par phase — Oren
- [x] Décision : Telegram uniquement — Oren
- [x] Création du dépôt et de la documentation v0 — Oren
