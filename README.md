# Driverss

Automatisation du dispatch de Driverss : prise de commande (sadran, puis agent vocal), backend qui garde la vérité, diffusion des courses aux chauffeurs sur Telegram (bot et Mini App).

> **Statut : Phase 0 — Validation.** Aucun code de production avant la porte **G0** (avis juridique favorable, accord signé avec Yossef, données reçues). État à jour : [`docs/CURRENT_STATE.md`](docs/CURRENT_STATE.md).

**En une ligne :** Eitan = conversation → Oren = vérité et logique → Ilan = diffusion et chauffeur → Oren = état final → client et sadran informés.

---

## Structure du dépôt

```
driverss/
├── AGENTS.md              ← cerveau commun des IA (à lire en premier)
├── CLAUDE.md              ← renvoie vers AGENTS.md pour Claude Code
├── README.md
├── docs/
│   ├── PRODUCT.md         ← ce qu'est Driverss, scope V1
│   ├── PLAN.md            ← plan détaillé par phase, qui fait quoi
│   ├── ARCHITECTURE.md    ← modules, responsables, schémas
│   ├── API_CONTRACTS.md   ← interfaces entre les 3 modules
│   ├── DATABASE.md        ← tables et règles de données
│   ├── MINIAPP.md         ← Mini App Telegram chauffeur : périmètre, écrans
│   ├── DECISIONS.md       ← décisions importantes et pourquoi
│   ├── CURRENT_STATE.md   ← où en est le projet maintenant
│   └── TASKS.md           ← maintenant / ensuite / plus tard / fait
├── backend/               ← Oren — Supabase, logique métier, source de vérité
├── voice/                 ← Eitan — téléphonie SIP, agent vocal ElevenLabs
├── dispatch/              ← Ilan — bot Telegram, Mini App chauffeur, dashboard du sadran
└── .github/               ← CODEOWNERS, modèle de pull request
```

Chaque module a son propre `AGENTS.md` (règles locales) et son `CLAUDE.md` (qui l'importe).

## Par où commencer

| Je veux… | Je lis |
| --- | --- |
| Comprendre le projet | [`PRODUCT.md`](docs/PRODUCT.md) puis [`PLAN.md`](docs/PLAN.md) |
| Savoir quoi faire aujourd'hui | [`CURRENT_STATE.md`](docs/CURRENT_STATE.md) puis [`TASKS.md`](docs/TASKS.md) |
| Coder dans un module | `AGENTS.md` du module + [`ARCHITECTURE.md`](docs/ARCHITECTURE.md) |
| Toucher une interface entre modules | [`API_CONTRACTS.md`](docs/API_CONTRACTS.md) |
| Toucher la base | [`DATABASE.md`](docs/DATABASE.md) |
| Travailler sur la Mini App chauffeur | [`MINIAPP.md`](docs/MINIAPP.md) + `dispatch/AGENTS.md` |
| Comprendre pourquoi on a choisi X | [`DECISIONS.md`](docs/DECISIONS.md) |

## Équipe

| Personne | Rôle | Module |
| --- | --- | --- |
| Oren | Responsable technique : backend, base, contrats d'API, sécurité, revue de code | `backend/` |
| Eitan | Produit et business : accord avec Yossef, agent vocal, vente aux stations | `voice/` |
| Ilan | Dispatch et opérations : bot Telegram, Mini App chauffeur, dashboard, pilote chauffeurs, tests de bout en bout | `dispatch/` |
| Papa | Terrain : relation quotidienne avec Yossef, chauffeurs, portes des stations | — |

## Travailler avec une IA (Claude Code ou autre)

1. L'IA lit `AGENTS.md` (racine), puis celui du module, puis `docs/CURRENT_STATE.md`.
2. Elle travaille uniquement dans le module de la personne qui la lance, sauf demande explicite.
3. En fin de session, elle met à jour `docs/CURRENT_STATE.md` et `docs/TASKS.md`. Toute décision importante va dans `docs/DECISIONS.md`.

### Plugins Claude Code à installer

Quatre plugins seulement, tous officiels ou de partenaires connus. Chacun les installe sur son poste, dans Claude Code : commande `/plugin`, puis chercher le nom.

| Plugin | Qui l'installe | Quand | Pourquoi |
| --- | --- | --- | --- |
| `context7` | Oren, Ilan, Eitan | Maintenant | Donne à Claude la documentation à jour des bibliothèques (Supabase, supabase-js, Telegram…), qui changent vite |
| `supabase` | Oren | Dès le prototype TMA-03 | Accès au projet Supabase et bonnes pratiques Postgres (RLS, migrations, fonctions SQL) |
| `security-guidance` | Oren, Ilan (Eitan s'il écrit du code) | Dès le premier code | Repère dans le code écrit par Claude les secrets en dur, failles d'authentification et injections |
| `playwright` | Ilan | Phase 1 | Pilote un vrai navigateur pour tester la Mini App et le dashboard de bout en bout (règle d'or 8) |

**Règles pour le plugin `supabase`** (il permet d'exécuter du SQL directement) : seulement sur un projet de **développement**, **jamais la production** ; en **lecture seule** et limité à ce projet ; approbation manuelle de chaque requête. Tout changement de schéma reste une migration dans git.

Déjà inclus dans Claude Code, rien à installer : `/code-review` (relire une PR) et `/security-review`.

Déconseillés : `telegram` (sert à piloter Claude Code depuis Telegram, pas à construire notre bot), `frontend-design` (pousse vers un design chargé, contraire à la spec de la Mini App), `superpowers` (impose un processus qui doublonne avec `AGENTS.md`).

## Workflow Git

- `main` est protégée : tout passe par une pull request.
- Branches : `oren/<sujet>`, `eitan/<sujet>`, `ilan/<sujet>`.
- Tout ce qui touche `backend/`, les contrats d'API ou la base est relu par Oren (voir `.github/CODEOWNERS`).
- Une fonctionnalité est « finie » seulement après un test de bout en bout où la donnée finale en base est correcte.

## Secrets et données

- Jamais de secret dans le dépôt : utiliser `.env` (ignoré par git), à partir de `.env.example`.
- Jamais de données clients, d'enregistrements d'appels ou d'export du logiciel de Yossef dans le dépôt. Le dossier `data/` est ignoré par git pour ça.
