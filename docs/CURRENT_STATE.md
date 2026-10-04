# État actuel

**Mis à jour le :** 4 octobre 2026
**Phase :** 0 — Validation (semaines 1 à 3)
**Prochaine porte :** G0 — avis juridique acceptable, accord signé, données reçues

## Où on en est

- Accord de principe oral avec Yossef. Rien de signé : ni parts, ni redevance, ni propriété du code.
- Plan détaillé écrit ([`PLAN.md`](PLAN.md)) et rôles répartis entre Oren, Eitan, Ilan et Papa.
- Dépôt créé, documentation v0 en place. **Aucun code.**
- Décision prise : Telegram uniquement pour les chauffeurs ([D-001](DECISIONS.md)).
- Spec v0 de la Mini App Telegram chauffeur rédigée ([`MINIAPP.md`](MINIAPP.md)) : architecture, authentification, temps réel, claim, contrats J à R, colonnes ajoutées. Décisions D-009 à D-016 **proposées**, pas encore relues par Ilan. Dossier `dispatch/miniapp/` réservé, sans code.

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
- Pas de 7 % pour une course annulée ou un client absent ([D-006](DECISIONS.md), à valider avec Yossef).
- Mini App chauffeur ([D-009 à D-016](DECISIONS.md)) : relecture par Ilan et Oren (TMA-01).
- Émission de la session des chauffeurs (D-011, option a ou b) : après le prototype TMA-03.
- Neuf questions ouvertes sur la Mini App, avec qui tranche : « Questions ouvertes » dans [`MINIAPP.md`](MINIAPP.md).

## Prochaines étapes

1. Réunion de lancement avec Yossef (Papa).
2. Trouver l'avocat et lui envoyer les questions (Eitan).
3. Recevoir l'export et l'analyser (Papa, puis Oren).
4. Test Telegram avec 10 chauffeurs, y compris l'ouverture d'une Mini App de démonstration (Ilan, avec Papa).
5. Relire la spec de la Mini App (Ilan, Oren), puis prototype jetable d'authentification et de temps réel (Oren).
