# AGENTS.md — cerveau commun

Ce fichier s'adresse à toute IA (Claude Code, Codex, Cursor…) qui travaille dans ce dépôt. Lis-le en entier avant d'agir.

## Contexte en 5 lignes

- Driverss est un service de courses et de livraisons commandées par téléphone, surtout dans le public orthodoxe. Fondateur : Yossef.
- Driverss prélève 12 % au chauffeur : 7 % sont crédités au client (ou donnés à une institution via Nedarim Plus), 5 % couvrent la technologie.
- Aujourd'hui, tout repose sur des sadranim humains : ils répondent, publient la course aux chauffeurs, ressaisissent tout dans un logiciel.
- On construit le système qui automatise ce travail : saisie unique, diffusion Telegram (bot et Mini App chauffeur), claim atomique, puis agent vocal.
- Le système doit pouvoir être revendu à d'autres stations, sous une marque neutre.

## Phase actuelle

**Phase 0 — Validation.** Pas de code de production avant la porte G0. Les prototypes jetables sont permis, dans une branche, clairement marqués comme tels. Vérifie toujours `docs/CURRENT_STATE.md` : il fait foi.

## Lire d'abord, dans cet ordre

1. `docs/CURRENT_STATE.md` — où on en est.
2. Le `AGENTS.md` du module dans lequel tu travailles.
3. `docs/API_CONTRACTS.md` si tu touches une interface, `docs/DATABASE.md` si tu touches la base, `docs/MINIAPP.md` si tu touches la Mini App chauffeur.
4. `docs/DECISIONS.md` avant de proposer un changement d'approche.

## Qui possède quoi

| Module | Responsable | Possède | Ne fait jamais |
| --- | --- | --- | --- |
| `backend/` | Oren | Base, prix et devis, statuts, transitions, claim atomique, éligibilité des chauffeurs, droits de chaque appelant, auth des chauffeurs (initData), temps réel, file des envois du bot, grand livre des 7 %, sécurité | Prompt vocal détaillé, UX du bot et de la Mini App |
| `voice/` | Eitan | Téléphonie SIP, agent ElevenLabs, prompt, tests vocaux, transfert humain | Calculer un prix, écrire en base, choisir un chauffeur |
| `dispatch/` | Ilan | Bot Telegram, Mini App chauffeur (`dispatch/miniapp/`), dashboard du sadran, repli manuel, mesures du pilote | Calculer un prix, créer une course dans son propre stockage, décider qui est notifié ou qui peut prendre une course, décider seul qu'une course est prise |

Tu travailles dans le module de la personne qui t'a lancé. Pour modifier un autre module, demande d'abord.

## Les règles d'or

1. **Le backend est la seule source de vérité.** `voice/` et `dispatch/` (bot, Mini App, dashboard) ne parlent qu'aux APIs du backend, jamais directement à Postgres.
2. **Le prix vient toujours du backend.** Personne n'invente, ne recalcule ni n'annonce un prix qui ne vient pas de l'API.
3. **Le claim est atomique côté backend.** Deux chauffeurs qui cliquent en même temps, dans le bot ou dans la Mini App : un seul gagne, et c'est le backend qui le décide.
4. **Un contrat d'API ne change pas en douce.** Toute modification passe par `docs/API_CONTRACTS.md`, dans une pull request relue par les deux côtés.
5. **Un humain toujours joignable.** On ne supprime jamais le transfert vers un sadran ni l'attribution manuelle.
6. **Telegram uniquement** pour les chauffeurs. Pas de WhatsApp, ni comme canal ni comme repli (décision D-001).
7. **`station_id` partout.** Chaque table métier porte la station, dès le premier jour.
8. **Une fonctionnalité est finie** seulement après un test de bout en bout où la donnée finale en base est correcte.

## Conventions

- **Langue :** documentation en français ; code, noms de tables, colonnes, variables et endpoints en anglais ; textes vus par les clients et chauffeurs en hébreu.
- **Argent :** entiers en agorot (`price_agorot`), jamais de nombres à virgule.
- **Dates :** stockées en `timestamptz` (UTC), affichées en heure d'Israël (`Asia/Jerusalem`). Format d'échange : ISO 8601 avec décalage.
- **Téléphones :** format E.164 (`+9725XXXXXXXX`).
- **Statuts de course :** `created`, `posted`, `claimed`, `done`, `cancelled`, `no_show` (voir `docs/DATABASE.md`).

## Interdits absolus

- Commiter un secret, une clé, un token ou un fichier `.env`.
- Commiter des données clients, des enregistrements d'appels ou un export du logiciel de Yossef.
- Mettre un numéro de téléphone ou un nom de client dans un message, un écran ou un signal temps réel vu par plusieurs chauffeurs.
- Mettre un secret (`service_role`, `x-driverss-key`, token du bot) ailleurs que côté serveur — jamais dans la Mini App ni dans le dashboard.
- Modifier la production à la main (console Supabase) au lieu d'une migration.

## Fin de session

Avant de rendre la main :

1. Mets à jour `docs/CURRENT_STATE.md` (ce qui a avancé, ce qui bloque).
2. Coche ou ajoute les tâches dans `docs/TASKS.md`.
3. Si une décision a été prise, ajoute une entrée dans `docs/DECISIONS.md`.
4. Si un contrat a changé, vérifie que `docs/API_CONTRACTS.md` est à jour.
