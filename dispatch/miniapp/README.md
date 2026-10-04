# dispatch/miniapp — Mini App Telegram chauffeur

**Responsable : Ilan.** Lis d'abord [`dispatch/AGENTS.md`](../AGENTS.md).

> **Dossier réservé : aucun code avant la porte G0** (D-002). Un prototype jetable, comme la page de démonstration du test TMA-02, vit dans une branche, marqué comme tel, et n'est jamais fusionné ici.

Une seule application monopage (D-010), mobile, en hébreu de droite à gauche, ouverte dans Telegram. Elle ne parle qu'au backend, par les contrats J à P.

| Pour… | Lire |
| --- | --- |
| Périmètre, parcours, écrans, états, textes | [`docs/MINIAPP.md`](../../docs/MINIAPP.md) |
| Ce que la Mini App ne fait jamais, auth, temps réel, claim | [`docs/ARCHITECTURE.md`](../../docs/ARCHITECTURE.md) |
| Requêtes, réponses, erreurs, canaux temps réel (J à P) | [`docs/API_CONTRACTS.md`](../../docs/API_CONTRACTS.md) |
| Avancement (`TMA-xx`) | [`docs/TASKS.md`](../../docs/TASKS.md) |

## Stack proposée (D-010, à confirmer par Ilan)

Vite, React et TypeScript ; supabase-js pour la session et le temps réel ; script officiel `telegram-web-app.js`. Pas de bibliothèque de carte en V1. Même stack que le dashboard. Hébergement statique en HTTPS, sur un domaine stable.

## Structure prévue (Phase 1)

```
dispatch/miniapp/
├── README.md
├── index.html
├── src/
│   ├── telegram/      ← initData, thème, MainButton, BackButton, retour au premier plan
│   ├── api/           ← client des contrats J à O, mode mock
│   ├── realtime/      ← canaux P, relecture, repli par polling
│   ├── screens/       ← courses disponibles, détail, mes courses, profil, états
│   └── i18n/          ← textes en hébreu
└── mocks/             ← réponses copiées des exemples de API_CONTRACTS.md
```

## Développer sans backend

Les exemples JSON des contrats J à O sont des réponses valides : ils servent de mocks. Un petit générateur local simule les signaux de P (nouvelle course, course prise). Passer des mocks au vrai backend ne change que la configuration.

## Configuration

Uniquement des valeurs publiques : l'URL du projet Supabase et la clé `anon`. **Jamais** `service_role`, `x-driverss-key` ni token du bot : ce code est envoyé sur le téléphone du chauffeur.
