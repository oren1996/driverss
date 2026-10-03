# voice/ — AGENTS.md

**Responsable : Eitan.** Lis d'abord le `AGENTS.md` à la racine.

## Mission

Transformer un appel téléphonique réel en une commande structurée, fiable et confirmée, en utilisant uniquement les APIs du backend. Le module ne possède ni le prix ni l'état d'une course.

## Calendrier

- **Phase 1 :** prototype hors production, sur les vrais enregistrements.
- **Phase 2 :** production sur la nuit et le débordement, puis sur tous les appels si les seuils de la porte G2 sont atteints.

## Organisation

```
voice/
├── agent/
│   ├── prompt.md          ← prompt système de l'agent, versionné ici
│   ├── tools.json         ← définitions des tools exportées d'ElevenLabs
│   └── settings.md        ← voix, langue, transfert, réglages de fin d'appel
├── dictionary/
│   └── places.md          ← villes, quartiers, synagogues, termes yiddish et leurs variantes
└── tests/
    └── scenarios.md       ← scénarios de test et résultats
```

L'agent est configuré dans l'interface ElevenLabs, mais **sa configuration est recopiée ici à chaque changement**, pour garder l'historique et pouvoir revenir en arrière.

## Règles

- L'agent n'annonce **jamais** un prix qui ne vient pas de `get-quote`.
- Il appelle seulement les tools du backend : `lookup-customer`, `get-quote`, `create-ride`, `GET /rides/{id}`.
- Il ne crée une course qu'après une confirmation explicite du client (« oui »), avec une `idempotency_key` par appel.
- Transfert vers un sadran : touche 0, demande du client, ou lieu toujours incompris après une relance.
- Il confirme l'adresse à voix haute avant de créer la course.
- Aucun secret dans le prompt.
- Pas d'enregistrement d'appel réel dans le dépôt : les fichiers de test restent hors git (`data/`).

## Scénarios de test minimum

Accents, bruit de fond, ville marmonnée, interruption, saisie au clavier, demande de transfert, client inconnu, course hors table de prix, appel pendant Shabbat.
