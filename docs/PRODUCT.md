# Produit

## Ce qu'est Driverss

Driverss est un service de courses et de livraisons commandées par téléphone, surtout dans le public orthodoxe. Le fondateur est Yossef. Un client appelle, un sadran publie la course aux chauffeurs, un chauffeur la prend.

Ce qui le distingue : **7 % de chaque course reviennent au client**, sur un solde cumulé. Avec le partenariat Nedarim Plus, ces 7 % peuvent aussi être donnés à une synagogue ou une institution choisie par le client (code d'institution).

## Le modèle économique annoncé par Yossef

- Driverss prélève **12 %** de chaque course au chauffeur.
- **7 %** sont crédités au client, ou donnés à son institution.
- **5 %** doivent couvrir la technologie.
- Le profit doit venir plus tard d'un **abonnement mensuel payé par les chauffeurs**, une fois que Driverss devient leur principale source de travail.

Données de départ annoncées : 756 clients inscrits, environ 30 courses par jour pour 40 à 50 appels. Ces chiffres sont à vérifier sur l'export du logiciel (Phase 0).

## Le problème qu'on résout

Tout repose sur des sadranim humains : répondre au téléphone, publier la course, trouver un chauffeur, ressaisir la course dans le logiciel, appeler les clients pour les avis, recouvrer les commissions. À volume élevé, ce coût humain dépasse la marge. L'automatisation est la condition pour que le modèle soit rentable.

## Les utilisateurs

| Utilisateur | Ce qu'il fait | Canal |
| --- | --- | --- |
| Client | Commande une course, cumule ses 7 % | Téléphone (074 ou numéro étoile) |
| Sadran | Saisit, suit, attribue à la main en cas de besoin | Dashboard web, sur mobile |
| Chauffeur | Reçoit les courses, clique « je prends » | Bot Telegram |
| Gabbaï / institution | Reçoit les dons de 7 % | Nedarim Plus (Phase 3 et après) |
| Yossef | Pilote l'activité | Dashboard |

## Scope

| V1 — Phases 1 et 2 | Plus tard — après validation |
| --- | --- |
| Saisie unique par le sadran dans le dashboard | Adresse complète entièrement automatisée |
| Table de prix déterministe (si validée en Phase 0) | Facturation des chauffeurs (Phase 3) |
| Course persistée, crédit de 7 % au statut « terminée » | Avis clients automatiques (Phase 3) |
| Diffusion Telegram et claim atomique | Parrainage et tirage au sort, avec consentement (Phase 3) |
| Repli : relance Telegram puis attribution manuelle | Application ou bot pour le grand public |
| Statuts : terminée, annulée, client absent | Ligne de consultation de solde |
| Agent vocal sur la nuit et le débordement, puis tout (Phase 2) | Multi-stations réel, marque neutre (Phase 4) |
| Transfert vers un sadran à tout moment | Agent de rappel, RAG avancé |
| Fermeture Shabbat et fêtes | |

## Hors scope, volontairement

- **WhatsApp** : les chauffeurs passent sur Telegram uniquement (décision D-001).
- Toute fonctionnalité qui repose sur le fait de rester invisible aux concurrents (la « ligne cachée »).

## Ce qui compte comme succès

- **Phase 1 :** toutes les courses passent par le système pendant deux semaines, aucune double attribution, soldes concordants.
- **Phase 2 :** au moins 85 % des commandes vocales complètes sans humain, moins de 2 % d'erreurs d'adresse (seuils à valider avec Yossef).
- **Phase 3 :** un mois de facturation sans erreur.
- **Phase 4 :** une station pilote qui paie.
