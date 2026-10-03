# Décisions

Une entrée par décision importante, la plus récente en haut. On n'efface pas une décision : on en ajoute une nouvelle qui la remplace.

Statuts : **Acceptée** (on l'applique), **Proposée** (à valider), **Remplacée** (voir la décision qui la remplace).

---

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
