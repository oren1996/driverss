# dispatch/ — AGENTS.md

**Responsable : Ilan.** Lis d'abord le `AGENTS.md` à la racine.

## Mission

Prendre une course déjà validée par le backend, notifier les chauffeurs que le backend désigne, leur donner une interface pour la prendre (bot et Mini App Telegram), et donner au sadran une vue en direct avec la possibilité d'attribuer à la main.

## Organisation

```
dispatch/
├── bot/            ← bot Telegram : inscription (partage du contact), envoi des courses depuis la file du backend, bouton « je prends », message privé au gagnant
├── miniapp/        ← Mini App Telegram chauffeur : une seule SPA (docs/MINIAPP.md)
└── dashboard/      ← application web du sadran : saisie, suivi en direct, attribution manuelle, gestion des chauffeurs
```

Spec de la Mini App : `docs/MINIAPP.md`. Frontières, auth, envois du bot, temps réel, claim : `docs/ARCHITECTURE.md`. Contrats : `docs/API_CONTRACTS.md` (D, E, Q, R et T pour le bot ; J à P pour la Mini App ; B, C, F, G, H et S pour le dashboard). Tâches : `SOC-xx` (socle) et `TMA-xx` (Mini App) dans `docs/TASKS.md`.

## Règles communes

- **Telegram uniquement.** Pas de WhatsApp (décision D-001). La Mini App vit dans Telegram : pas de version web publique.
- Le bot, la Mini App et le dashboard ne calculent **jamais** un prix, ne créent **jamais** de course dans leur propre stockage, et ne décident **jamais** qui est notifié, qui voit ni qui prend une course : ils utilisent les contrats du backend.
- Personne ne décide seul qu'une course est prise : le bot (E), la Mini App (O) et le dashboard (F) affichent la réponse du backend.
- Le message de course reprend **exactement** le format que les chauffeurs utilisent aujourd'hui (fourni par le backend dans `message_text`).
- **Jamais de numéro ni de nom de client** dans un message, un écran ou une liste vus par plusieurs chauffeurs. Les détails de prise en charge vont seulement au chauffeur attribué : message privé (envoi R), écran « mes courses » de la Mini App.
- Le webhook Telegram vérifie l'en-tête secret envoyé par Telegram.
- Repli : course non prise en 60 secondes → relance déclenchée par le backend, avec des destinataires recalculés → surlignée dans le dashboard pour attribution manuelle.
- Le dashboard et la Mini App sont pensés pour mobile, en hébreu, de droite à gauche. Leur code tourne chez l'utilisateur : clé `anon` + jeton de l'utilisateur seulement, jamais `service_role`, `x-driverss-key` ni token du bot.

## Règles propres au dashboard

- Il ne fait que ce que le rôle du sadran permet (D-017) : un `dispatcher` saisit, attribue et clôt ; un `admin` gère en plus les chauffeurs (S). La station vient de la session, jamais d'un choix à l'écran.
- La saisie passe toujours par un devis (B) puis `create-ride` (C) avec son `quote_id` ; si le devis a expiré, refaire le devis et redemander l'accord du client.
- Pour clore ou annuler, envoyer la version affichée (`expected_version`, G) ; sur `version_conflict`, relire la course et laisser le sadran décider.
- Mettre en évidence les courses relancées encore libres, les envois échoués et, avant Shabbat, les courses encore libres.

## Règles propres à la Mini App

- Une seule SPA (D-010), qui ne parle qu'aux contrats J à P.
- À l'ouverture, elle échange l'initData contre une session (J). Session en mémoire seulement ; jamais d'initData ni de jeton dans l'URL, les logs ou le stockage local.
- `initDataUnsafe` sert à l'affichage, jamais à décider.
- Les signaux temps réel ne sont que des déclencheurs : l'état affiché vient toujours d'une relecture (M, N). Une réponse plus ancienne (`version`) que ce qui est affiché est ignorée. Le repli par polling est obligatoire.
- Aucune donnée client en cache persistant.
- Le bouton « התקשר לסדרן » reste toujours accessible (règle d'or 5).
- Ni carte, ni filtres, ni paiement, ni abonnement en Phase 1 (D-015).

## Règles propres au bot

- Inscription : bouton de partage du contact ; vérifier que `contact.user_id` égale `from.id` ; normaliser en E.164 ; appeler Q.
- N'envoyer que ce que la file du backend contient (contrat T, D-019) : lire, envoyer, confirmer chaque envoi avec l'identifiant du message Telegram. Jamais d'envoi décidé par le bot, jamais de minuteur dans le bot : la relance vient du backend.
- Envois D (`ride_posted`, `ride_relaunch`) : message privé au format habituel, avec « אני לוקח » (appelle E) et « פתח » (ouvre la Mini App sur la course). Envois R : message au gagnant avec les détails, modification des messages des autres (« נלקחה », « בוטלה »), message au chauffeur dont la course est annulée.
- Limites de Telegram : moins de 30 messages par seconde au total, au plus un par seconde au même chauffeur. Sur un refus 429, confirmer `retry` avec le délai indiqué par Telegram.
- Pendant Shabbat et les fêtes, la file reste vide (D-020) : le bot n'envoie rien.
- Le bouton du bot reste un chemin de claim complet : la porte G1 ne dépend pas de la Mini App.

## Tests obligatoires avant fusion

- Deux chauffeurs cliquent en même temps, l'un dans le bot, l'autre dans la Mini App : un seul gagne, l'autre voit « déjà prise ».
- Course non prise : la relance part à 60 secondes, la course est surlignée dans le dashboard.
- Le bot redémarre au milieu d'une diffusion : seuls les envois non confirmés repartent.
- Course annulée avant le message au gagnant : il ne part pas.
- Refus 429 de Telegram : l'envoi attend le délai indiqué, puis repart.
- Aucun numéro ni nom de client dans les messages de diffusion, la liste de la Mini App ou les signaux temps réel.
- Dashboard : un `dispatcher` ne peut pas gérer les chauffeurs ; un devis expiré est refusé ; une annulation sur une course modifiée entre-temps est refusée (`version_conflict`).
- Mini App : initData falsifié ou trop vieux → refus ; chauffeur non lié ou bloqué → le bon écran ; temps réel coupé → la liste se met à jour par polling.
- Dashboard et Mini App utilisables sur un petit écran de téléphone, de droite à gauche.

## Mesures du pilote (chaque semaine)

Délai avant claim, part des courses non prises en 60 secondes, doubles attributions, part des claims par canal (bot, Mini App, dashboard), envois échoués, échecs d'ouverture de la Mini App, plaintes des chauffeurs, chauffeurs qui n'ont pas pu installer Telegram.
