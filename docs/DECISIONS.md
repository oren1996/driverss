# Décisions

Une entrée par décision importante, la plus récente en haut. On n'efface pas une décision : on en ajoute une nouvelle qui la remplace.

Statuts : **Acceptée** (on l'applique), **Proposée** (à valider), **Remplacée** (voir la décision qui la remplace).

---

## D-016 — Contrats : l'identité vient de l'authentification ; un seul format d'erreur

- **Date :** 4 octobre 2026 · **Statut :** Proposée — à confirmer par Oren, à relire par Ilan
- **Décision :** aucun contrat n'accepte l'identité de l'acteur dans le corps de la requête. Le chauffeur vient de son jeton (Mini App) ou du `telegram_user_id` reçu par le webhook vérifié du bot : `claim-ride` n'envoie plus de `driver_id`. Le sadran vient de sa session : `assign-ride` et `update-ride-status` perdent `assigned_by` et `actor_id`. Toutes les erreurs suivent `{ "ok": false, "error": { "code", "message" } }`, y compris `claim-ride`, qui utilisait `reason`.
- **Pourquoi :** avec la Mini App, des requêtes partent directement des téléphones ; un identifiant dans le corps se falsifie. Le bot n'avait de toute façon aucun moyen de connaître `driver_id`. Deux formats d'erreur auraient obligé chaque client à gérer les deux.

## D-015 — Périmètre de la Mini App en Phase 1 : ni carte, ni filtres, ni abonnement, ni paiement

- **Date :** 4 octobre 2026 · **Statut :** Proposée — à valider par Ilan et Oren
- **Décision :** en Phase 1, la Mini App comprend : connexion Telegram, profil, disponibilité simple, courses disponibles, détail, claim, mes courses, état de la connexion ([`MINIAPP.md`](MINIAPP.md)). Pas en Phase 1 : carte et carte de la demande, zones, rayon, GPS, filtres, offres directes, historique, gains, abonnement, paiement, blocage pour dette.
- **Pourquoi :** la porte G1 mesure « 100 % des courses dans le système, 0 double attribution », pas la richesse de l'interface. Oren est le goulot d'étranglement ; avec environ 30 courses par jour et quelques dizaines de chauffeurs, ni filtres ni zones ne se justifient. Abonnement et paiement relèvent de la Phase 3, et la dette passera par `drivers.status = blocked`, déjà prévu.

## D-014 — Éligibilité calculée par le backend ; notification en message privé ; disponibilité simple

- **Date :** 4 octobre 2026 · **Statut :** Proposée — à valider par Ilan (diffusion) et Papa (règle de relance)
- **Décision :** le backend calcule qui est notifié (`recipients` dans `ride_ready_for_dispatch` : chauffeurs actifs, Telegram lié, disponibles), qui voit une course et qui peut la prendre (chauffeur actif de la station). Le bot envoie un message privé à chaque destinataire. La disponibilité est un interrupteur avec une fin optionnelle (24 h au plus) : deux colonnes sur `drivers`, vraie par défaut, qui ne filtrent que les notifications. La relance à 60 s va aux mêmes destinataires, puis le sadran attribue. Pas de PostGIS, de zones ni de table d'offres en Phase 1.
- **Pourquoi :** l'éligibilité est une règle métier (règle d'or 1). Un groupe Telegram ne permet ni de respecter la disponibilité, ni d'exclure tout de suite un chauffeur bloqué, ni d'ouvrir la Mini App par un bouton `web_app` (réservé aux conversations privées). Disponible par défaut : au lancement, le comportement reste celui d'avant (« tous les inscrits »). Respecter « pas disponible » à la relance protège la confiance dans l'interrupteur ; le sadran peut toujours appeler un chauffeur.
- **Précise :** le repli de D-001 (« relance à tous les chauffeurs inscrits ») devient « relance aux chauffeurs disponibles ».

## D-013 — Claim instantané et atomique, une seule fonction pour tous les canaux

- **Date :** 4 octobre 2026 · **Statut :** Proposée — à confirmer par Oren (prolonge D-004)
- **Décision :** pas de « demande » soumise à l'accord d'un sadran : le premier claim valide gagne. Le bot (`claim-ride`), la Mini App (`driver-claim-ride`) et le dashboard (`assign-ride`) appellent la même fonction SQL `claim_ride` : une requête conditionnelle `posted → claimed`, l'événement écrit dans la même transaction, idempotente pour le même chauffeur (`replay`), refus journalisés (`claim_rejected`). Les détails de prise en charge ne sont renvoyés qu'au chauffeur attribué, tant que la course est `claimed`. Pas d'heure d'arrivée estimée en Phase 1.
- **Pourquoi :** automatiser le travail du sadran est la raison d'être du projet ; une approbation humaine recrée le goulot. Une seule fonction donne une seule règle de concurrence, quel que soit le canal. L'idempotence couvre le double appui et la réponse perdue sur un réseau mobile faible. Les refus journalisés tranchent les litiges (« j'ai cliqué le premier »).
- **Plus tard :** un mode « demande + approbation » par station, si une station l'exige (Phase 4), sans toucher au mode par défaut.

## D-012 — Temps réel de la Mini App : signaux Broadcast privés, relecture par l'API, repli par polling

- **Date :** 4 octobre 2026 · **Statut :** Proposée — à confirmer par Oren
- **Décision :** un trigger sur `rides` émet des signaux sans donnée (`ride_id`, type) sur des canaux privés `rides:{station_id}` et `driver:{driver_id}`, autorisés par RLS sur `realtime.messages`. À chaque signal, la Mini App relit l'état par l'API. Sans abonnement actif, elle relit toutes les 20 s. Pas de `postgres_changes` pour les chauffeurs.
- **Pourquoi :** `postgres_changes` envoie la ligne (adresse, client) à tout abonné qui passe le filtre de ligne. Le « signal + relecture » garde une seule source de données — l'API, qui applique l'éligibilité — et aucune donnée personnelle dans le canal. Les filtres des téléphones casher peuvent bloquer les WebSockets : le polling est obligatoire, pas optionnel. À 30 courses par jour, la relecture ne coûte rien.

## D-011 — Authentification de la Mini App : l'initData échangé une fois contre une session courte

- **Date :** 4 octobre 2026 · **Statut :** Proposée — à confirmer par Oren après le prototype TMA-03
- **Décision :** la Mini App envoie l'initData une seule fois, à `driver-auth-telegram`. Le backend vérifie la signature et la fraîcheur (`auth_date` de moins de 5 minutes), retrouve le chauffeur pré-inscrit et lié par (`station_id`, `telegram_user_id`), et renvoie une session Supabase courte (jeton d'une heure au plus, plus un jeton de rafraîchissement, gardés en mémoire). Les chauffeurs n'ont aucun accès direct aux tables : tout passe par les fonctions `driver-*`, et leur seule politique RLS porte sur la réception temps réel. Aucun chauffeur n'est créé à la connexion.
- **Pourquoi :** envoyer l'initData brut à chaque requête oblige soit à accepter un initData vieux de plusieurs heures (il n'est pas renouvelé tant que la Mini App reste ouverte), soit à casser la session ; et le temps réel privé comme RLS demandent un JWT. Une session Supabase apporte le rafraîchissement et la révocation sans code maison. Pas de création à la volée : sinon, n'importe quel compte Telegram deviendrait chauffeur.
- **Option de repli :** un JWT signé par le backend, si l'émission d'une session Supabase côté serveur pose problème au prototype.

## D-010 — Une seule application monopage pour la Mini App

- **Date :** 4 octobre 2026 · **Statut :** Proposée — Ilan choisit la stack
- **Décision :** la Mini App est une seule SPA statique (proposition : Vite, React, TypeScript, supabase-js, script officiel de Telegram), avec la même stack que le dashboard. Pas de pages rendues par un serveur à côté, pas de deuxième application.
- **Pourquoi :** une application concurrente étudiée mêle une SPA récente et plusieurs pages serveur héritées : deux façons de s'authentifier, deux designs, deux bases de code. Une seule SPA = une seule authentification, un seul client d'API, un seul endroit à tester. La même stack que le dashboard : Ilan n'apprend qu'une chose. Pas de serveur à maintenir : hébergement statique.

## D-009 — Une Mini App Telegram pour les chauffeurs, dans `dispatch/`, en plus du bot

- **Date :** 4 octobre 2026 · **Statut :** Proposée — à valider par Ilan et Oren
- **Décision :** les chauffeurs ont une Mini App Telegram (`dispatch/miniapp/`, responsable Ilan), en **complément** du bot. En Phase 1, **le bot est le canal principal** : inscription, notification de chaque course, claim complet en un clic, message privé au gagnant, message mis à jour quand la course est prise. La Mini App ajoute la vue d'ensemble (liste en direct, mes courses, disponibilité). Elle arrive en Phase 1 après le bot ; la porte G1 n'en dépend pas, et elle peut glisser en Phase 2 si le temps manque. Le backend reste la seule source de vérité (D-004).
- **Pourquoi :** l'étude en lecture seule d'une application concurrente, depuis un compte chauffeur autorisé, montre que le bot y reste le canal principal : toutes les courses arrivent en message privé, on peut demander une course depuis le message sans ouvrir la Mini App, et la Mini App n'a aucune notification. Elle sert surtout à régler la disponibilité et à voir l'état de ses demandes. Elle reste dans Telegram, donc D-001 est respecté. Le bot reste aussi le chemin garanti si des téléphones filtrés ou de vieilles versions de Telegram n'ouvrent pas la Mini App (tâche TMA-02).

## D-008 — L'argent est stocké en agorot

- **Date :** 3 octobre 2026 · **Statut :** Proposée — à confirmer par Oren
- **Décision :** tous les montants sont des entiers en agorot (`price_agorot`).
- **Pourquoi :** 7 % de 180 ₪ = 12,60 ₪. Les nombres à virgule créent des erreurs d'arrondi dans un grand livre.

## D-007 — Aucune donnée client dans le dépôt

- **Date :** 3 octobre 2026 · **Statut :** Acceptée · **Par :** tous
- **Décision :** ni export du logiciel de Yossef, ni enregistrements d'appels, ni numéros de clients dans git. Le dossier `data/` est ignoré.
- **Pourquoi :** amendement 13 de la loi sur la vie privée, en vigueur depuis le 14 août 2025, et un dépôt git ne s'efface jamais vraiment.

## D-006 — Le crédit de 7 % et la commission partent seulement à « terminée »

- **Date :** 3 octobre 2026 · **Statut :** Proposée — à valider avec Yossef
- **Décision :** rien au grand livre pour une course annulée ou un client absent.
- **Pourquoi :** sinon on crédite des courses qui n'ont pas eu lieu.

## D-005 — `station_id` dans chaque table dès le premier jour

- **Date :** 3 octobre 2026 · **Statut :** Proposée — à confirmer par Oren
- **Décision :** chaque table métier porte la station, même s'il n'y en a qu'une au début.
- **Pourquoi :** le système doit pouvoir être revendu à d'autres stations (Phase 4). L'ajouter plus tard coûterait une migration douloureuse.

## D-004 — Le backend est la seule source de vérité ; claim atomique

- **Date :** 3 octobre 2026 · **Statut :** Acceptée · **Par :** tous
- **Décision :** voix et dispatch ne parlent qu'aux APIs du backend. Le claim et l'attribution manuelle sont une même opération atomique côté backend.
- **Pourquoi :** éviter les doubles attributions et les prix inventés.

## D-003 — La voix sort du chemin critique

- **Date :** 3 octobre 2026 · **Statut :** Acceptée · **Par :** tous
- **Décision :** la Phase 1 livre la saisie unique, le bot et le dashboard, sans voix. L'agent vocal est prototypé en parallèle, puis branché en Phase 2 sur la nuit et le débordement.
- **Pourquoi :** la reconnaissance vocale (hébreu orthodoxe, yiddish, noms de rues) est le composant le plus risqué. Le lancement Nedarim Plus peut arriver avant que la voix soit prête.

## D-002 — Aucun code de production avant la porte G0

- **Date :** 3 octobre 2026 · **Statut :** Acceptée · **Par :** tous
- **Décision :** pas de production avant un avis juridique acceptable, un accord signé avec Yossef et la réception des données.
- **Pourquoi :** le transport de passagers par des chauffeurs privés est illégal en Israël, et rien n'est encore signé.

## D-001 — Telegram uniquement pour les chauffeurs

- **Date :** 3 octobre 2026 · **Statut :** Acceptée · **Par :** Oren
- **Décision :** les courses sont diffusées aux chauffeurs uniquement par un bot Telegram. Plus de WhatsApp, ni comme canal principal ni comme repli.
- **Repli :** course non prise en 60 secondes → relance à tous les chauffeurs inscrits → attribution manuelle par le sadran. Les chauffeurs sans Telegram reçoivent leurs courses par téléphone, via le sadran.
- **Risque accepté :** des chauffeurs aux téléphones filtrés peuvent ne pas pouvoir installer Telegram. Le test de la Phase 0 le mesurera ; si une grosse partie ne peut pas passer, on rouvre cette décision.
- **Précisée par :** D-009 (une Mini App Telegram s'ajoute au bot) et D-014 (relance aux chauffeurs disponibles).
