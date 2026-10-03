# dispatch/ — AGENTS.md

**Responsable : Ilan.** Lis d'abord le `AGENTS.md` à la racine.

## Mission

Prendre une course déjà validée par le backend, la diffuser aux chauffeurs sur Telegram, récupérer un claim, et donner au sadran une vue en direct avec la possibilité d'attribuer à la main.

## Organisation

```
dispatch/
├── bot/            ← bot Telegram : inscription des chauffeurs, diffusion, bouton « je prends »
└── dashboard/      ← application web du sadran : saisie, suivi en direct, attribution manuelle
```

## Règles

- **Telegram uniquement.** Pas de WhatsApp (décision D-001).
- Le bot et le dashboard ne calculent **jamais** un prix et ne créent **jamais** de course dans leur propre stockage : ils utilisent `create-ride`, `claim-ride`, `assign-ride`, `update-ride-status` du backend.
- Le bot ne décide pas seul qu'une course est prise : il appelle `claim-ride` et affiche la réponse (`claimed` ou `already_taken`).
- Le message de course reprend **exactement** le format que les chauffeurs utilisent aujourd'hui (fourni par le backend dans `message_text`).
- **Jamais de numéro ni de nom de client** dans un message vu par plusieurs chauffeurs. Les détails de prise en charge vont seulement au gagnant, en message privé.
- Le webhook Telegram vérifie l'en-tête secret envoyé par Telegram.
- Repli : course non prise en 60 secondes → relance à tous les chauffeurs inscrits → surlignée dans le dashboard pour attribution manuelle.
- Le dashboard est pensé pour mobile, en hébreu, de droite à gauche. Il utilise la clé `anon` de Supabase + RLS, jamais la clé `service_role`.

## Tests obligatoires avant fusion

- Deux chauffeurs cliquent en même temps : un seul gagne, l'autre voit « déjà prise ».
- Course non prise : la relance part à 60 secondes, la course est surlignée dans le dashboard.
- Aucun numéro client dans les messages de groupe.
- Dashboard utilisable sur un petit écran de téléphone.

## Mesures du pilote (chaque semaine)

Délai avant claim, part des courses non prises en 60 secondes, doubles attributions, plaintes des chauffeurs, chauffeurs qui n'ont pas pu installer Telegram.
