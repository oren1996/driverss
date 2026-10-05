# dispatch/ — AGENTS.md

**Responsable : Ilan.** Lis d'abord le `AGENTS.md` à la racine.

## Mission

Prendre une course déjà validée par le backend, notifier les chauffeurs que le backend désigne, leur donner une interface pour la prendre (bot et Mini App Telegram), et donner au sadran une vue en direct avec la possibilité d'attribuer à la main.

## Organisation

```
dispatch/
├── bot/            ← bot Telegram : inscription (partage du contact), notification, bouton « je prends », message privé au gagnant
├── miniapp/        ← Mini App Telegram chauffeur : une seule SPA (docs/MINIAPP.md)
└── dashboard/      ← application web du sadran : saisie, suivi en direct, attribution manuelle
```

Spec de la Mini App : `docs/MINIAPP.md`. Frontières, auth, temps réel, claim : `docs/ARCHITECTURE.md`. Contrats : `docs/API_CONTRACTS.md` (D, E, Q, R pour le bot ; J à P pour la Mini App ; B, C, F, G, H pour le dashboard). Tâches : `TMA-xx` dans `docs/TASKS.md`.

## Règles communes

- **Telegram uniquement.** Pas de WhatsApp (décision D-001). La Mini App vit dans Telegram : pas de version web publique.
- Le bot, la Mini App et le dashboard ne calculent **jamais** un prix, ne créent **jamais** de course dans leur propre stockage, et ne décident **jamais** qui est notifié, qui voit ni qui prend une course : ils utilisent les contrats du backend.
- Personne ne décide seul qu'une course est prise : le bot (E), la Mini App (O) et le dashboard (F) affichent la réponse du backend.
- Le message de course reprend **exactement** le format que les chauffeurs utilisent aujourd'hui (fourni par le backend dans `message_text`).
- **Jamais de numéro ni de nom de client** dans un message, un écran ou une liste vus par plusieurs chauffeurs. Les détails de prise en charge vont seulement au chauffeur attribué : message privé à la réception de R, écran « mes courses » de la Mini App.
- Le webhook Telegram vérifie l'en-tête secret envoyé par Telegram.
- Repli : course non prise en 60 secondes → relance aux mêmes destinataires → surlignée dans le dashboard pour attribution manuelle.
- Le dashboard et la Mini App sont pensés pour mobile, en hébreu, de droite à gauche. Leur code tourne chez l'utilisateur : clé `anon` + jeton de l'utilisateur seulement, jamais `service_role`, `x-driverss-key` ni token du bot.

## Règles propres à la Mini App

- Une seule SPA (D-010), qui ne parle qu'aux contrats J à P.
- À l'ouverture, elle échange l'initData contre une session (J). Session en mémoire seulement ; jamais d'initData ni de jeton dans l'URL, les logs ou le stockage local.
- `initDataUnsafe` sert à l'affichage, jamais à décider.
- Les signaux temps réel ne sont que des déclencheurs : l'état affiché vient toujours d'une relecture (M, N). Le repli par polling est obligatoire.
- Aucune donnée client en cache persistant.
- Le bouton « התקשר לסדרן » reste toujours accessible (règle d'or 5).
- Ni carte, ni filtres, ni paiement, ni abonnement en Phase 1 (D-015).

## Règles propres au bot

- Inscription : bouton de partage du contact ; vérifier que `contact.user_id` égale `from.id` ; normaliser en E.164 ; appeler Q.
- Notifier uniquement les `recipients` de D, en message privé, avec « אני לוקח » (appelle E) et « פתח » (ouvre la Mini App sur la course).
- À la réception de R : message privé au gagnant avec les détails, « נלקחה » sur les messages des autres, message au chauffeur dont la course est annulée. Ignorer un `event_id` déjà traité.
- Le bouton du bot reste un chemin de claim complet : la porte G1 ne dépend pas de la Mini App.

## Tests obligatoires avant fusion

- Deux chauffeurs cliquent en même temps, l'un dans le bot, l'autre dans la Mini App : un seul gagne, l'autre voit « déjà prise ».
- Course non prise : la relance part à 60 secondes, la course est surlignée dans le dashboard.
- Aucun numéro ni nom de client dans les messages de diffusion, la liste de la Mini App ou les signaux temps réel.
- Mini App : initData falsifié ou trop vieux → refus ; chauffeur non lié ou bloqué → le bon écran ; temps réel coupé → la liste se met à jour par polling.
- Dashboard et Mini App utilisables sur un petit écran de téléphone, de droite à gauche.

## Mesures du pilote (chaque semaine)

Délai avant claim, part des courses non prises en 60 secondes, doubles attributions, part des claims par canal (bot, Mini App, dashboard), échecs d'ouverture de la Mini App, plaintes des chauffeurs, chauffeurs qui n'ont pas pu installer Telegram.
