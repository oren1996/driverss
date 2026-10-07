# dispatch/ — AGENTS.md

**Responsable : Ilan.** Lis d'abord le `AGENTS.md` à la racine.

## Mission

Prendre une course déjà validée par le backend, notifier les chauffeurs que le backend désigne, leur donner une interface pour la prendre (bot et Mini App Telegram), et donner au sadran une vue en direct avec la possibilité d'attribuer à la main.

## Organisation

```
dispatch/
├── bot/            ← bot Telegram : inscription (partage du contact), envois de la file du backend, bouton « je prends », message privé au gagnant
├── miniapp/        ← Mini App Telegram chauffeur : une seule SPA (docs/MINIAPP.md)
└── dashboard/      ← application web du sadran : saisie, suivi en direct, attribution manuelle, gestion des chauffeurs
```

Spec de la Mini App : `docs/MINIAPP.md`. Frontières, auth, envois du bot, temps réel, claim : `docs/ARCHITECTURE.md`. Contrats : `docs/API_CONTRACTS.md` (D, E, Q, R et T pour le bot ; J à P pour la Mini App ; B, C, F, G, H et S pour le dashboard). Tâches : `SOC-xx` (socle) et `TMA-xx` (Mini App) dans `docs/TASKS.md`.

## Règles communes

- **Telegram uniquement.** Pas de WhatsApp (décision D-001). La Mini App vit dans Telegram : pas de version web publique.
- Le bot, la Mini App et le dashboard ne calculent **jamais** un prix, ne créent **jamais** de course dans leur propre stockage, et ne décident **jamais** qui est notifié, qui voit ni qui prend une course : ils utilisent les contrats du backend.
- Personne ne décide seul qu'une course est prise : le bot (E), la Mini App (O) et le dashboard (F) affichent la réponse du backend.
- Le message de course reprend **exactement** le format que les chauffeurs utilisent aujourd'hui (fourni par le backend dans `message_text`).
- **Jamais de numéro ni de nom de client** dans un message, un écran ou une liste vus par plusieurs chauffeurs. Les détails de prise en charge vont seulement au chauffeur attribué : message privé (annonce R `ride_assigned`), écran « mes courses » de la Mini App. Un message modifié n'en contient jamais.
- Le webhook Telegram vérifie l'en-tête secret envoyé par Telegram.
- Repli : course non prise en 60 secondes → relance déclenchée par le backend, avec des destinataires recalculés → surlignée dans le dashboard pour attribution manuelle.
- Le dashboard et la Mini App sont pensés pour mobile, en hébreu, de droite à gauche. Leur code tourne chez l'utilisateur : clé `anon` + jeton de l'utilisateur seulement, jamais `service_role`, `x-driverss-key` ni token du bot.

## Règles propres au dashboard

- Il ne fait que ce que le rôle du sadran permet (D-017) : un `dispatcher` saisit, attribue et clôt ; un `admin` gère en plus les chauffeurs (S) et corrige le grand livre. La station vient de la session, jamais d'un choix à l'écran.
- La saisie passe toujours par un devis (B) puis `create-ride` (C) avec son `quote_id` ; si le devis a expiré, refaire le devis et redemander l'accord du client.
- Pour clore ou annuler, envoyer la version affichée (`expected_version`, G) ; sur `version_conflict`, relire la course et laisser le sadran décider.
- Mettre en évidence les courses relancées encore libres et, avant Shabbat, les courses encore libres.
- Montrer les signalements (`dispatcher_alerts`) jusqu'à ce qu'un sadran les marque traités : attribution ou avis d'annulation non remis (appeler le chauffeur), chauffeur qui a bloqué le bot, chauffeur délié avec des courses en cours.
- Délier un chauffeur avec la bonne raison : `changed_account` ou `compromised` (D-022). Après un vol, n'autoriser une nouvelle liaison (`allow_relink`) qu'au téléphone avec le chauffeur, et l'ancien compte Telegram seulement s'il a fermé les autres sessions.
- Corrections du grand livre : `admin` seulement, chacune avec son `correction_id` (D-018), pour qu'un double envoi n'écrive rien de plus.

## Règles propres à la Mini App

- Une seule SPA (D-010), qui ne parle qu'aux contrats J à P.
- À l'ouverture, elle échange l'initData contre une session (J). Session en mémoire seulement ; jamais d'initData ni de jeton dans l'URL, les logs ou le stockage local.
- `initDataUnsafe` sert à l'affichage, jamais à décider.
- Sur `driver_not_linked` (jamais lié, ou liaison révoquée depuis l'ouverture), oublier la session et afficher l'écran « non inscrit ».
- Les signaux temps réel ne sont que des déclencheurs : l'état affiché vient toujours d'une relecture (M, N). Une réponse plus ancienne (`version`) que ce qui est affiché est ignorée. Le repli par polling est obligatoire.
- Aucune donnée client en cache persistant.
- Le bouton « התקשר לסדרן » reste toujours accessible (règle d'or 5).
- Ni carte, ni filtres, ni paiement, ni abonnement en Phase 1 (D-015).

## Règles propres au bot

- Inscription : bouton de partage du contact ; vérifier que `contact.user_id` égale `from.id` ; normaliser en E.164 ; appeler Q. Sur `link_requires_admin`, dire au chauffeur d'appeler le sadran.
- N'envoyer que ce que la file du backend contient (contrat T, D-019) : lire, envoyer, confirmer. Jamais d'envoi décidé par le bot, jamais de minuteur dans le bot : la relance vient du backend.
- Lire peu d'envois à la fois et les envoyer aussitôt. Ne rien commencer après `send_before`, qui tient compte de la prochaine fermeture ; appels à Telegram de 10 s au plus. Un délai dépassé ne prouve pas que Telegram n'a rien fait : répondre `unknown`. Un envoi non tenté à temps est rendu par `retry` avec `retry_after_s: 0`.
- Confirmer juste après la réponse de Telegram : `sent` avec l'identifiant du message (aussi quand Telegram répond que le message modifié est déjà identique) ; `retry` sur un refus 429, avec le délai indiqué ; `failed` sur un refus définitif (`bot_blocked`, `chat_not_found`, `message_unavailable`) ; `unknown` quand il n'y a pas de réponse claire (délai dépassé, connexion coupée, erreur 5xx). Jamais `retry` ni `failed` si Telegram n'a pas clairement refusé : le message est peut-être parti.
- Annonces : `ride_posted` et `ride_relaunch` (D), message privé au format habituel, avec « אני לוקח » (appelle E) et « פתח » (ouvre la Mini App sur la course) ; `ride_assigned`, message privé au gagnant avec les détails.
- Corrections : `message_update` modifie le message `edit_message_id` selon `state` (`yours`, `taken`, `cancelled`), sans bouton ni détails de prise en charge ; `ride_cancelled` envoie « הנסיעה בוטלה » au chauffeur attribué, avec le trajet et l'heure. Les textes de chaque état sont ceux du bot, en hébreu.
- Secours : quand un chauffeur appuie sur le bouton d'une course qui n'est plus libre, afficher la réponse de E et corriger ce message-là. Ce secours ne remplace pas les corrections : un chauffeur qui ne clique pas doit quand même voir l'état juste.
- Limites de Telegram : moins de 30 messages par seconde au total, au plus un par seconde au même chauffeur, modifications comprises.
- Pendant Shabbat et les fêtes, la file ne remet rien (D-020) : le bot n'envoie rien.
- Le bouton du bot reste un chemin de claim complet : la porte G1 ne dépend pas de la Mini App.

## Tests obligatoires avant fusion

Ce sont des tests fonctionnels : ils seront écrits et exécutés pendant l'implémentation (Phase 1). Aujourd'hui, seule la documentation existe.

- Deux chauffeurs cliquent en même temps, l'un dans le bot, l'autre dans la Mini App : un seul gagne, l'autre voit « déjà prise ».
- Course non prise : la relance part à 60 secondes, la course est surlignée dans le dashboard.
- Le bot redémarre au milieu d'une diffusion : seuls les envois non confirmés repartent.
- Chauffeur qui a reçu la diffusion et la relance, course prise par un autre : ses deux messages sont corrigés ; le message du gagnant aussi, que la course ait été prise par le bot, par la Mini App ou attribuée par le sadran.
- Course prise puis terminée pendant que le bot est arrêté : au redémarrage, les messages sont quand même corrigés.
- **Envoi sans confirmation suivi d'une annulation :** arrêter le bot juste après l'envoi d'une attribution, avant la confirmation ; annuler la course ; redémarrer. Le chauffeur reçoit « הנסיעה בוטלה », pas une seconde attribution ; 2 minutes plus tard, l'avis repart une seconde fois.
- **Claim dans la Mini App, puis annulation avant le message d'attribution :** prendre la course dans la Mini App, la fermer, annuler avant que le bot ait lu l'attribution. Le chauffeur reçoit quand même « הנסיעה בוטלה » dans Telegram.
- **Confirmation tardive :** retarder la confirmation d'une diffusion au-delà de 60 s pendant qu'un autre chauffeur prend la course : le message finit corrigé.
- **Corrections reçues dans le désordre :** course prise puis annulée pendant que les corrections partent : chaque message finit sur l'état final, « בוטלה ». Si une ancienne modification arrive en retard, la vérification remet l'état final 2 minutes plus tard.
- Le bot ne commence aucun envoi après `send_before` ; sur un délai dépassé, il répond `unknown`, jamais `retry`.
- Refus 429 de Telegram : l'envoi attend le délai indiqué, puis repart.
- **Fermeture Shabbat :** rien ne part pendant la fermeture, pas même un envoi lu juste avant ; à la sortie, les corrections partent.
- **Tentative de réassociation de l'ancien compte :** après une déliaison `compromised`, le partage du contact depuis l'ancien compte reçoit « appelle le sadran » ; son bouton « אני לוקח » ne prend plus aucune course.
- Aucun numéro ni nom de client dans les messages de diffusion, les messages modifiés, la liste de la Mini App ou les signaux temps réel.
- Dashboard : un `dispatcher` ne peut pas gérer les chauffeurs ni corriger le grand livre ; un devis expiré est refusé ; une annulation sur une course modifiée entre-temps est refusée (`version_conflict`) ; un signalement reste visible jusqu'à ce qu'il soit traité.
- Mini App : initData falsifié ou trop vieux → refus ; chauffeur non lié ou bloqué → le bon écran ; **ancien jeton après une déliaison → écran « non inscrit »** ; temps réel coupé → la liste se met à jour par polling.
- Dashboard et Mini App utilisables sur un petit écran de téléphone, de droite à gauche.

## Mesures du pilote (chaque semaine)

Délai avant claim, part des courses non prises en 60 secondes, doubles attributions, part des claims par canal (bot, Mini App, dashboard), envois échoués ou peut-être partis, signalements traités, échecs d'ouverture de la Mini App, plaintes des chauffeurs, chauffeurs qui n'ont pas pu installer Telegram.
