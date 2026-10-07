# État actuel

**Mis à jour le :** 7 octobre 2026
**Phase :** 0 — Validation (semaines 1 à 3)
**Prochaine porte :** G0 — avis juridique acceptable, accord signé, données reçues

## Où on en est

- Accord de principe oral avec Yossef. Rien de signé : ni parts, ni redevance, ni propriété du code.
- Plan détaillé écrit ([`PLAN.md`](PLAN.md)) et rôles répartis entre Oren, Eitan, Ilan et Papa.
- Dépôt créé, documentation v0 en place. **Aucun code.**
- Décision prise : Telegram uniquement pour les chauffeurs ([D-001](DECISIONS.md)).
- Spec v0 de la Mini App chauffeur fusionnée (PR #1, relue par Ilan). Ses décisions sont acceptées, sauf D-011 (après le prototype TMA-03) et la règle de relance de D-014.
- Relecture critique de la documentation le 7 octobre : droits des appelants, devis, envois du bot, grand livre, concurrence et planning corrigés. Nouvelles propositions D-017 à D-021.
- Deuxième relecture le même jour : les messages du bot sont suivis un par un et corrigés (annonces et corrections, envois « peut-être partis »), délier un chauffeur coupe l'accès de l'ancien téléphone tout de suite, les corrections du grand livre sont possibles sans réécrire l'historique, les droits SQL sont explicites. Nouvelle proposition D-022 ; D-018 à D-020 précisées. En attente de la relecture d'Ilan (PR #2).
- Le dépôt GitHub est public et `main` n'est pas protégée : à régler (Oren). Seul Oren est collaborateur du dépôt : Ilan et Eitan sont à ajouter.

## Ce qui bloque

| Blocage | Qui débloque |
| --- | --- |
| Pas d'avis juridique sur le transport de passagers | Eitan (trouver l'avocat) |
| Pas d'accord écrit avec Yossef | Eitan, avec Papa |
| Pas encore de données (export, enregistrements, messages des groupes) | Papa |
| Heures disponibles de chacun inconnues : le calendrier est une estimation | Tous |

## Décisions en attente

- Source de vérité des soldes de 7 % : le logiciel de Yossef ou le nôtre.
- Modèle de prix : table fixe ou négociation dans le groupe.
- Lancement Nedarim Plus : le décaler jusqu'à la fin de la Phase 1, ou démarrer avec les sadranim humains seuls.
- Pas de 7 % pour une course annulée ou un client absent ([D-006](DECISIONS.md), avec Yossef).
- Session des chauffeurs (D-011, option a ou b) : après le prototype TMA-03.
- Relance aux seuls chauffeurs disponibles (D-014) : Papa.
- Droits et comptes des sadranim (D-017) : Oren et Ilan.
- Règle d'arrondi du grand livre (D-018) : avec Yossef, à vérifier sur l'export.
- File des envois du bot (D-019) : Oren et Ilan.
- Aucun envoi pendant Shabbat et les fêtes (D-020) : Papa et Oren.
- Devis identifié et version des courses (D-021) : Oren ; Eitan à prévenir (la voix crée aussi des courses).
- Liaisons Telegram, déliaison et procédure après un vol (D-022) : Oren et Ilan.
- Course close par erreur : corriger aussi son statut, ou seulement le grand livre ? Avec Yossef, avant le pilote.
- Questions sur la Mini App : « Questions ouvertes » dans [`MINIAPP.md`](MINIAPP.md).

## Prochaines étapes

1. Ajouter Ilan et Eitan comme collaborateurs du dépôt GitHub, puis le passer en privé (Oren).
2. Réunion de lancement avec Yossef (Papa).
3. Trouver l'avocat et lui envoyer les questions (Eitan).
4. Recevoir l'export et l'analyser, dont la règle d'arrondi (Papa, puis Oren).
5. Test Telegram avec 10 chauffeurs, y compris l'ouverture d'une Mini App de démonstration (Ilan, avec Papa).
6. Relecture de la PR #2 par Ilan, puis trancher D-017 à D-022 ; ensuite, prototype jetable d'authentification et de temps réel (Oren, avec Ilan).
