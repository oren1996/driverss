# Décisions

Une entrée par décision importante, la plus récente en haut. On n'efface pas une décision : on en ajoute une nouvelle qui la remplace.

Statuts :

- **Acceptée** : on l'applique.
- **Proposée** : hypothèse de travail. On peut concevoir avec, mais rien d'irréversible (code de production, migration, annonce aux chauffeurs ou à Yossef) ne se construit dessus avant qu'elle soit acceptée.
- **Remplacée** : voir la décision qui la remplace.

---

## D-021 — Le prix enregistré est celui du devis accepté ; chaque course porte une version

- **Date :** 7 octobre 2026 · **Statut :** Proposée — à confirmer par Oren ; Eitan à prévenir (la voix crée aussi des courses)
- **Décision :** `get-quote` renvoie un devis identifié (`quote_id`), valable 15 minutes. `create-ride` exige ce `quote_id` et reprend son prix, son trajet et son horaire ; un devis expiré est refusé (`quote_expired`) : nouveau devis et nouvel accord du client. Chaque course porte une `version`, +1 à chaque changement de statut, transmise dans les envois du bot, les signaux temps réel et les lectures. `update-ride-status` accepte `expected_version` et refuse (`version_conflict`) si la course a changé depuis que le sadran l'a vue.
- **Pourquoi :** sans référence au devis, `create-ride` recalculait le prix : si la table changeait entre l'annonce et la création, le client payait un autre prix que celui qu'il avait accepté. L'heure ne permet pas d'ordonner les changements : `now()` donne l'heure du début de la transaction, pas celle de sa validation. Et l'ordre des transactions ne protège pas une annulation : `claimed → cancelled` est permis, donc une annulation pensée pour une course libre pouvait s'appliquer à une course prise entre-temps.

## D-020 — Fermetures : aucun envoi aux chauffeurs pendant Shabbat et les fêtes

- **Date :** 7 octobre 2026 · **Statut :** Proposée — à valider par Papa (ce qui est acceptable pour les chauffeurs) et Oren
- **Décision :** pendant une fermeture (table `closures`), le backend ne crée ni envoi ni relance, et la file des envois ne renvoie rien au bot. Les envois en attente à l'entrée sont réévalués à la sortie : ceux qui sont dépassés sont abandonnés. Avant l'entrée, le dashboard signale les courses encore libres. S'applique dès la Phase 1.
- **Pourquoi :** le service s'adresse surtout au public orthodoxe, et beaucoup de chauffeurs en font partie : une notification pendant Shabbat serait mal reçue et pousserait à couper le bot. La fermeture automatique n'était prévue qu'en Phase 2, pour la voix ; or le bot diffuse dès la Phase 1.

## D-019 — Envois du bot : une file durable tenue par le backend

- **Date :** 7 octobre 2026 · **Statut :** Proposée — à valider par Oren et Ilan
- **Décision :**
  1. Dans la même transaction que chaque changement de statut, le backend crée un envoi par destinataire dans une file (`notifications`) : nouvelle course (D), relance, changement de statut (R).
  2. Le backend réveille le bot (appel sans données) et le refait régulièrement par sécurité. Le bot lit la file (contrat T), envoie, et confirme chaque envoi avec l'identifiant du message Telegram ; un envoi non confirmé en 60 s redevient disponible.
  3. Le backend ne remet jamais un envoi dépassé : si la `version` de la course a changé depuis sa création (D-021), l'envoi est abandonné.
  4. La relance à 60 s est une tâche planifiée du backend, qui recalcule les destinataires à ce moment-là. Une seule relance par course.
  5. Le bot respecte les limites de Telegram. Sur un refus 429, l'envoi est remis en file après le délai indiqué ; après 5 échecs, il est abandonné et signalé au sadran.
- **Pourquoi :** « livraison au moins une fois » ne disait pas comment reprendre une diffusion interrompue, ni comment éviter un « c'est à vous » après une annulation arrivée dans le désordre. Une file en base survit aux redémarrages et suit chaque destinataire. Un minuteur dans le bot se perdrait à son redémarrage. Recalculer à la relance évite de notifier un chauffeur bloqué ou devenu indisponible entre-temps. Le bot reste simple — lire, envoyer, confirmer — et ne parle qu'aux APIs du backend (règle d'or 1). En plus, le backend sait quand chaque chauffeur a été prévenu, ce qui mesure le délai entre la notification et le claim.
- **Écartée :** garder l'appel direct du backend vers le bot et donner au bot sa propre mémoire des envois : plus de logique dans le bot, et une exception à la règle d'or 1.
- **Modifie :** le transport des contrats D et R (leur contenu reste) et le repli de D-014 (relance déclenchée par le backend, plus par le bot).

## D-018 — Grand livre : règle d'arrondi et une seule écriture par course

- **Date :** 7 octobre 2026 · **Statut :** Proposée — à valider avec Yossef (mode d'arrondi de son logiciel)
- **Décision :** la commission (12 %) et le crédit (7 %) sont calculés chacun en agorot entières, une seule fois, au devis, puis verrouillés avec la course. La part technologie est la différence (commission − crédit), donc le total tombe toujours juste. Le mode d'arrondi est celui du logiciel de Yossef, vérifié sur l'export. Au passage à `done`, une seule écriture par course et par compte, garantie par la base. Une correction est une écriture inverse qui pointe vers l'écriture corrigée. Si notre base devient la source de vérité des soldes, l'import ouvre chaque solde par une écriture `opening_balance`, sans recréditer les courses passées.
- **Pourquoi :** stocker en agorot n'évite pas l'arrondi : 7 % de 19,90 ₪ font 139,3 agorot. Si notre arrondi diffère de celui de Yossef, la porte G1 (« soldes concordants ») échoue. L'unicité en base empêche un double crédit même si le passage à `done` est rejoué.

## D-017 — Trois types d'appelants et un tableau des droits ; les sadranim ont un compte et un rôle

- **Date :** 7 octobre 2026 · **Statut :** Proposée — à valider par Oren et Ilan
- **Décision :** trois types d'appelants : les modules (`x-driverss-key`), les sadranim (session Supabase, table `dispatchers`, rôle `dispatcher` ou `admin`) et les chauffeurs (jeton du contrat J). Chaque fonction déclare ceux qu'elle accepte, dans le tableau « Qui peut appeler quoi » de [`API_CONTRACTS.md`](API_CONTRACTS.md) ; l'identité et la station viennent toujours de l'authentification. Un `dispatcher` saisit, attribue et clôt les courses ; un `admin` gère en plus les chauffeurs (contrat S : créer, modifier, bloquer, débloquer, délier Telegram). Les comptes des sadranim sont créés par Oren, jamais par inscription publique.
- **Pourquoi :** la règle « deux familles d'appelants, jamais mélangées » oubliait le dashboard, alors que `get-quote` et `create-ride` servent à la fois la voix et le dashboard. Sans tableau unique, deux développeurs implémenteraient des droits différents. La gestion des chauffeurs était rangée dans la Mini App (ancienne question Q6), alors qu'elle fait partie du socle : sans elle, aucun chauffeur ne peut s'inscrire, même par le bot.

## D-016 — Contrats : l'identité vient de l'authentification ; un seul format d'erreur

- **Date :** 4 octobre 2026 · **Statut :** Acceptée le 7 octobre 2026 · **Par :** Oren et Ilan (PR #1)
- **Décision :** aucun contrat n'accepte l'identité de l'acteur dans le corps de la requête. Le chauffeur vient de son jeton (Mini App) ou du `telegram_user_id` reçu par le webhook vérifié du bot : `claim-ride` n'envoie plus de `driver_id`. Le sadran vient de sa session : `assign-ride` et `update-ride-status` perdent `assigned_by` et `actor_id`. Toutes les erreurs suivent `{ "ok": false, "error": { "code", "message" } }`, y compris `claim-ride`, qui utilisait `reason`.
- **Pourquoi :** avec la Mini App, des requêtes partent directement des téléphones ; un identifiant dans le corps se falsifie. Le bot n'avait de toute façon aucun moyen de connaître `driver_id`. Deux formats d'erreur auraient obligé chaque client à gérer les deux.
- **Complétée par :** D-017 (trois types d'appelants et tableau des droits).

## D-015 — Périmètre de la Mini App en Phase 1 : ni carte, ni filtres, ni abonnement, ni paiement

- **Date :** 4 octobre 2026 · **Statut :** Acceptée le 7 octobre 2026 · **Par :** Oren et Ilan (PR #1)
- **Décision :** en Phase 1, la Mini App comprend : connexion Telegram, profil, disponibilité simple, courses disponibles, détail, claim, mes courses, état de la connexion ([`MINIAPP.md`](MINIAPP.md)). Pas en Phase 1 : carte et carte de la demande, zones, rayon, GPS, filtres, offres directes, historique, gains, abonnement, paiement, blocage pour dette.
- **Pourquoi :** la porte G1 mesure « 100 % des courses dans le système, 0 double attribution », pas la richesse de l'interface. Oren est le goulot d'étranglement ; avec environ 30 courses par jour et quelques dizaines de chauffeurs, ni filtres ni zones ne se justifient. Abonnement et paiement relèvent de la Phase 3, et la dette passera par `drivers.status = blocked`, déjà prévu.

## D-014 — Éligibilité calculée par le backend ; notification en message privé ; disponibilité simple

- **Date :** 4 octobre 2026 · **Statut :** Proposée — Ilan a validé la diffusion en message privé (PR #1) ; reste la règle de relance (Papa)
- **Décision :** le backend calcule qui est notifié (envois « nouvelle course », contrat D : chauffeurs actifs, Telegram lié, disponibles), qui voit une course et qui peut la prendre (chauffeur actif de la station). Le bot envoie un message privé à chaque destinataire. La disponibilité est un interrupteur avec une fin optionnelle (24 h au plus) : deux colonnes sur `drivers`, vraie par défaut, qui ne filtrent que les notifications. La relance à 60 s est déclenchée par le backend, qui recalcule les destinataires à ce moment-là (D-019) ; ensuite, le sadran attribue. Pas de PostGIS, de zones ni de table d'offres en Phase 1.
- **Pourquoi :** l'éligibilité est une règle métier (règle d'or 1). Un groupe Telegram ne permet ni de respecter la disponibilité, ni d'exclure tout de suite un chauffeur bloqué, ni d'ouvrir la Mini App par un bouton `web_app` (réservé aux conversations privées). Disponible par défaut : au lancement, le comportement reste celui d'avant (« tous les inscrits »). Respecter « pas disponible » à la relance protège la confiance dans l'interrupteur ; le sadran peut toujours appeler un chauffeur.
- **Précise :** le repli de D-001 (« relance à tous les chauffeurs inscrits ») devient « relance aux chauffeurs disponibles ».

## D-013 — Claim instantané et atomique, une seule fonction pour tous les canaux

- **Date :** 4 octobre 2026 · **Statut :** Acceptée le 7 octobre 2026 · **Par :** Oren et Ilan (PR #1)
- **Décision :** pas de « demande » soumise à l'accord d'un sadran : le premier claim valide gagne. Le bot (`claim-ride`), la Mini App (`driver-claim-ride`) et le dashboard (`assign-ride`) appellent la même fonction SQL `claim_ride` : une requête conditionnelle `posted → claimed`, l'événement écrit dans la même transaction, idempotente pour le même chauffeur (`replay`), refus journalisés (`claim_rejected`). Les détails de prise en charge ne sont renvoyés qu'au chauffeur attribué, tant que la course est `claimed`. Pas d'heure d'arrivée estimée en Phase 1.
- **Pourquoi :** automatiser le travail du sadran est la raison d'être du projet ; une approbation humaine recrée le goulot. Une seule fonction donne une seule règle de concurrence, quel que soit le canal. L'idempotence couvre le double appui et la réponse perdue sur un réseau mobile faible. Les refus journalisés tranchent les litiges (« j'ai cliqué le premier »).
- **Plus tard :** un mode « demande + approbation » par station, si une station l'exige (Phase 4), sans toucher au mode par défaut.

## D-012 — Temps réel de la Mini App : signaux Broadcast privés, relecture par l'API, repli par polling

- **Date :** 4 octobre 2026 · **Statut :** Acceptée le 7 octobre 2026 · **Par :** Oren et Ilan (PR #1)
- **Décision :** un trigger sur `rides` émet des signaux sans donnée (`ride_id`, type) sur des canaux privés `rides:{station_id}` et `driver:{driver_id}`, autorisés par RLS sur `realtime.messages`. À chaque signal, la Mini App relit l'état par l'API. Sans abonnement actif, elle relit toutes les 20 s. Pas de `postgres_changes` pour les chauffeurs.
- **Pourquoi :** `postgres_changes` envoie la ligne (adresse, client) à tout abonné qui passe le filtre de ligne. Le « signal + relecture » garde une seule source de données — l'API, qui applique l'éligibilité — et aucune donnée personnelle dans le canal. Les filtres des téléphones casher peuvent bloquer les WebSockets : le polling est obligatoire, pas optionnel. À 30 courses par jour, la relecture ne coûte rien.

## D-011 — Authentification de la Mini App : l'initData échangé une fois contre une session courte

- **Date :** 4 octobre 2026 · **Statut :** Proposée — à confirmer par Oren après le prototype TMA-03
- **Décision :** la Mini App envoie l'initData une seule fois, à `driver-auth-telegram`. Le backend vérifie la signature et la fraîcheur (`auth_date` de moins de 5 minutes), retrouve le chauffeur pré-inscrit et lié par (`station_id`, `telegram_user_id`), et renvoie une session Supabase courte (jeton d'une heure au plus, plus un jeton de rafraîchissement, gardés en mémoire). Les chauffeurs n'ont aucun accès direct aux tables : tout passe par les fonctions `driver-*`, et leur seule politique RLS porte sur la réception temps réel. Aucun chauffeur n'est créé à la connexion.
- **Pourquoi :** envoyer l'initData brut à chaque requête oblige soit à accepter un initData vieux de plusieurs heures (il n'est pas renouvelé tant que la Mini App reste ouverte), soit à casser la session ; et le temps réel privé comme RLS demandent un JWT. Une session Supabase apporte le rafraîchissement et la révocation sans code maison. Pas de création à la volée : sinon, n'importe quel compte Telegram deviendrait chauffeur.
- **Option de repli :** un JWT signé par le backend, si l'émission d'une session Supabase côté serveur pose problème au prototype.

## D-010 — Une seule application monopage pour la Mini App

- **Date :** 4 octobre 2026 · **Statut :** Acceptée le 7 octobre 2026 (une seule SPA ; la stack reste au choix d'Ilan) · **Par :** Oren et Ilan (PR #1)
- **Décision :** la Mini App est une seule SPA statique (proposition : Vite, React, TypeScript, supabase-js, script officiel de Telegram), avec la même stack que le dashboard. Pas de pages rendues par un serveur à côté, pas de deuxième application.
- **Pourquoi :** une application concurrente étudiée mêle une SPA récente et plusieurs pages serveur héritées : deux façons de s'authentifier, deux designs, deux bases de code. Une seule SPA = une seule authentification, un seul client d'API, un seul endroit à tester. La même stack que le dashboard : Ilan n'apprend qu'une chose. Pas de serveur à maintenir : hébergement statique.

## D-009 — Une Mini App Telegram pour les chauffeurs, dans `dispatch/`, en plus du bot

- **Date :** 4 octobre 2026 · **Statut :** Acceptée le 7 octobre 2026 · **Par :** Oren et Ilan (PR #1)
- **Décision :** les chauffeurs ont une Mini App Telegram (`dispatch/miniapp/`, responsable Ilan), en **complément** du bot. En Phase 1, **le bot est le canal principal** : inscription, notification de chaque course, claim complet en un clic, message privé au gagnant, message mis à jour quand la course est prise. La Mini App ajoute la vue d'ensemble (liste en direct, mes courses, disponibilité). Elle arrive en Phase 1 après le bot ; la porte G1 n'en dépend pas, et elle peut glisser en Phase 2 si le temps manque. Le backend reste la seule source de vérité (D-004).
- **Pourquoi :** l'étude en lecture seule d'une application concurrente, depuis un compte chauffeur autorisé, montre que le bot y reste le canal principal : toutes les courses arrivent en message privé, on peut demander une course depuis le message sans ouvrir la Mini App, et la Mini App n'a aucune notification. Elle sert surtout à régler la disponibilité et à voir l'état de ses demandes. Elle reste dans Telegram, donc D-001 est respecté. Le bot reste aussi le chemin garanti si des téléphones filtrés ou de vieilles versions de Telegram n'ouvrent pas la Mini App (tâche TMA-02).

## D-008 — L'argent est stocké en agorot

- **Date :** 3 octobre 2026 · **Statut :** Acceptée le 7 octobre 2026 · **Par :** Oren
- **Décision :** tous les montants sont des entiers en agorot (`price_agorot`).
- **Pourquoi :** des entiers évitent toute question de décimales, dans la base comme dans le JSON et le JavaScript. Un `float` crée des erreurs d'arrondi ; `numeric` serait exact aussi, mais plus lourd à manipuler côté clients. Les entiers ne dispensent pas d'une règle d'arrondi pour les pourcentages (7 % de 180 ₪ = 12,60 ₪, mais 7 % de 19,90 ₪ = 1,393 ₪) : voir D-018.

## D-007 — Aucune donnée client dans le dépôt

- **Date :** 3 octobre 2026 · **Statut :** Acceptée · **Par :** tous
- **Décision :** ni export du logiciel de Yossef, ni enregistrements d'appels, ni numéros de clients dans git. Le dossier `data/` est ignoré.
- **Pourquoi :** amendement 13 de la loi sur la vie privée, en vigueur depuis le 14 août 2025, et un dépôt git ne s'efface jamais vraiment.

## D-006 — Le crédit de 7 % et la commission partent seulement à « terminée »

- **Date :** 3 octobre 2026 · **Statut :** Proposée — à valider avec Yossef
- **Décision :** rien au grand livre pour une course annulée ou un client absent.
- **Pourquoi :** sinon on crédite des courses qui n'ont pas eu lieu.

## D-005 — `station_id` dans chaque table dès le premier jour

- **Date :** 3 octobre 2026 · **Statut :** Acceptée le 7 octobre 2026 · **Par :** Oren
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
