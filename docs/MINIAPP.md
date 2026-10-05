# Mini App chauffeur

> **Spec v0 — proposée le 4 octobre 2026, à relire par Ilan.** Responsable : Ilan (produit, écrans, code dans [`dispatch/miniapp/`](../dispatch/miniapp/)). Contrats et sécurité : Oren. **Aucun code avant la porte G0** (D-002). Décisions : D-009 à D-016 dans [`DECISIONS.md`](DECISIONS.md).

La Mini App est un complément du bot, ouvert dans Telegram : elle donne au chauffeur la vue d'ensemble (courses disponibles, mes courses, disponibilité). Elle affiche ce que le backend lui donne et lui transmet les demandes du chauffeur ; elle ne décide rien. En Phase 1, le bot reste le canal principal : il notifie, et permet de prendre une course sans ouvrir la Mini App (D-009).

| Je cherche… | Je lis |
| --- | --- |
| Frontières, authentification, temps réel, claim, visibilité des données | [`ARCHITECTURE.md`](ARCHITECTURE.md) |
| Contrats J à P (Mini App), Q et R (bot) | [`API_CONTRACTS.md`](API_CONTRACTS.md) |
| Colonnes ajoutées pour la Mini App | [`DATABASE.md`](DATABASE.md) |
| Tâches `TMA-xx` | [`TASKS.md`](TASKS.md) |

## Périmètre

| Phase 1 | Plus tard — après G1, selon les mesures du pilote |
| --- | --- |
| Connexion automatique par Telegram, sans mot de passe | Disponibilité par zone, rayon ou trajet ; GPS |
| Profil : nom, téléphone, statut, bouton « appeler le sadran » | Carte, carte de la demande |
| Disponibilité : un interrupteur, avec fin optionnelle | Filtres : véhicule, prix, mots-clés, villes |
| Courses disponibles, mises à jour en direct | Offres directes à un chauffeur |
| Détail d'une course, claim en un appui | Historique, gains, commission due, export |
| Mes courses, avec les détails de prise en charge | Abonnement, paiement, blocage pour dette (Phase 3) |
| État de la connexion : en direct, mise à jour périodique, hors ligne | Préférences de notification |

## Parcours

**Première fois**

1. Le chauffeur est pré-inscrit avec son téléphone (import des données de Yossef, ou dashboard).
2. Il ouvre le bot et appuie sur « שלח את מספר הטלפון » : le bot lie son compte Telegram (contrat Q).
3. Le bot lui montre le bouton « פתח את האפליקציה ».

Si la Mini App est ouverte avant la liaison : écran « עדיין לא נרשמת » avec un bouton vers le bot.

**Une nouvelle course**

1. Le bot envoie un message privé au format habituel, avec deux boutons : « אני לוקח » (claim direct dans le bot) et « פתח » (la Mini App s'ouvre sur la course).
2. Si la Mini App est déjà ouverte, la course apparaît dans « נסיעות פנויות ».
3. Le chauffeur ouvre la course et appuie sur le bouton principal « אני לוקח ».
4. Gagnant : écran « הנסיעה שלך! » avec l'adresse, le téléphone du client (appel en un appui) et les notes. Le bot lui envoie aussi les détails en message privé.
5. Perdant : « הנסיעה כבר נלקחה », retour à la liste. Le plus souvent, la carte avait déjà disparu grâce au signal temps réel.

**Course attribuée par le sadran :** message privé du bot et signal ; la course apparaît dans « הנסיעות שלי ».

**Course annulée :** si elle était au chauffeur, bandeau « הנסיעה בוטלה » et message privé ; elle quitte « הנסיעות שלי ». Si elle était disponible, elle disparaît de la liste.

**Disponibilité :** l'interrupteur « זמין / לא זמין » est toujours visible. En passant à « זמין », choix rapide : sans fin, pendant 2 heures, jusqu'à minuit. L'interrupteur ne change que les notifications du bot : la liste reste visible et le claim reste possible.

## Navigation

Trois onglets en bas, de droite à gauche :

1. **נסיעות פנויות** — courses disponibles, écran d'accueil ;
2. **הנסיעות שלי** — mes courses, avec un badge du nombre ;
3. **פרופיל** — profil.

En-tête permanent : interrupteur de disponibilité et pastille de connexion. Le détail d'une course s'ouvre par-dessus la liste ; le bouton retour de Telegram (`BackButton`) y ramène, et le bouton principal de Telegram (`MainButton`) porte « אני לוקח ».

La disponibilité n'a pas d'onglet : un seul interrupteur ne justifie pas un écran, et il doit rester visible partout.

## Écrans

**Carte de course (liste)**

- trajet : ville et quartier de départ → ville et quartier d'arrivée ;
- horaire en heure d'Israël : « היום 08:00 » ou « מחר 08:00 », plus « בעוד 25 דק׳ » si c'est proche ;
- type (נוסעים / משלוח) et nombre de passagers ;
- prix en shekels, formaté depuis `price_agorot` (formatage seulement, aucun calcul) ;
- depuis combien de temps la course est publiée.

Jamais sur une carte : adresse, téléphone, nom, notes.

**Détail avant claim :** la carte, plus `message_text` (le format que les chauffeurs connaissent) et le bouton principal « אני לוקח ». Pas de boîte de confirmation : la course se joue en secondes ; le pilote mesurera les claims par erreur. Pendant l'appel, le bouton affiche un chargement et n'envoie rien de plus (un second envoi recevrait de toute façon `replay`).

**Détail après claim (`driver_view = mine`) :** en plus, l'adresse de prise en charge (copier, lien « פתח ב-Waze »), le téléphone du client (appel en un appui), les notes du sadran et le bouton « התקשר לסדרן ».

**Mes courses :** mes courses `claimed`, par heure de prise en charge ; chacune ouvre le détail complet.

**Profil :** nom, téléphone, station, statut, disponibilité, « התקשר לסדרן », version de l'application.

Ce qu'un chauffeur peut voir avant et après le claim est fixé par le backend : tableau « Ce que voit un chauffeur » dans [`ARCHITECTURE.md`](ARCHITECTURE.md).

## États

| État | Quand | Ce qu'on affiche |
| --- | --- | --- |
| Chargement | Ouverture, échange de l'initData | Squelette de la liste |
| Non inscrit | `driver_not_linked` | « עדיין לא נרשמת » + bouton vers le bot |
| Bloqué | `driver_blocked` | « החשבון מושהה » + « התקשר לסדרן » |
| Session à refaire | `init_data_expired`, `invalid_init_data`, rafraîchissement refusé | « סגור ופתח מחדש » |
| Hors ligne | Pas de réseau | Bandeau « אין חיבור לאינטרנט » ; bouton « אני לוקח » désactivé |
| Mise à jour périodique | Canal temps réel non abonné | Pastille orange ; relecture toutes les 20 s |
| En direct | Canal temps réel abonné | Pastille verte |
| Liste vide | Aucune course disponible | « אין נסיעות פנויות כרגע » |
| Erreur inattendue | Erreur 5xx | « משהו השתבש » + « נסה שוב » |

## Textes clés

À valider avec Papa et les 10 chauffeurs testeurs (vocabulaire, ton).

| Usage | Texte |
| --- | --- |
| Onglet courses disponibles | נסיעות פנויות |
| Onglet mes courses | הנסיעות שלי |
| Onglet profil | פרופיל |
| Bouton de claim | אני לוקח |
| Claim réussi | הנסיעה שלך! |
| Claim perdu | הנסיעה כבר נלקחה |
| Course plus disponible | הנסיעה כבר לא זמינה |
| Course annulée | הנסיעה בוטלה |
| Disponibilité | זמין / לא זמין |
| Liste vide | אין נסיעות פנויות כרגע |
| Appeler le sadran | התקשר לסדרן |

## Hébreu, RTL, mobile

- `<html lang="he" dir="rtl">`, propriétés CSS logiques (`margin-inline-start`…), jamais `left` ni `right`.
- Nombres, prix, heures et téléphones isolés (`<bdi>` ou `dir="ltr"`) pour qu'ils ne s'inversent pas.
- Heure d'Israël (`Asia/Jerusalem`), format 24 h. Prix avec `Intl.NumberFormat('he-IL', { style: 'currency', currency: 'ILS' })`.
- Couleurs du thème Telegram (`themeParams`, clair et sombre) : pas d'identité visuelle copiée d'un concurrent, pas de marque en dur (`brand_name` vient de la station).
- Texte de 16 px au moins, zones tactiles de 44 px au moins, contraste élevé : beaucoup de chauffeurs ont des téléphones anciens.
- Application légère : pas de carte ni de grosse bibliothèque en Phase 1.
- Composants Telegram : `ready()`, `expand()`, `MainButton`, `BackButton`, `HapticFeedback` (succès, échec) ; relecture au retour au premier plan.

## Contraintes Telegram à connaître

- Les boutons `web_app` ne marchent que dans une conversation privée avec le bot : c'est une des raisons de notifier en message privé (D-014). Depuis un groupe, il faudrait un lien direct `t.me/<bot>/<app>?startapp=ride_153`.
- Le bouton « פתח » porte l'identifiant de la course dans l'URL de la Mini App (ou dans `start_param` pour un lien direct). C'est de la navigation : le backend vérifie toujours les droits.
- L'initData est créé à l'ouverture et n'est jamais renouvelé : la Mini App l'échange tout de suite contre une session (contrat J).
- Téléphones filtrés et vieilles versions de Telegram : la Mini App peut ne pas s'ouvrir, ou le temps réel être bloqué. À mesurer en Phase 0 (TMA-02). Le bouton du bot reste le chemin garanti ; le polling, le repli du temps réel.

## Mesures du pilote

- Part des claims par canal : bot, Mini App, dashboard (`ride_events.payload.via`).
- Échecs d'ouverture et d'authentification, par code.
- Délai entre la notification et le claim.
- Claims par erreur signalés au sadran.
- Chauffeurs pour qui la Mini App ne s'ouvre pas.

## Leçons de l'étude d'une application concurrente

Étude en lecture seule, depuis un compte chauffeur autorisé, de ce qu'un chauffeur voit et vit. Rien de son code n'est repris.

| Observé chez le concurrent | Ce qu'on en tire |
| --- | --- |
| Toutes les courses arrivent en message privé du bot ; on peut demander une course depuis le message, sans la Mini App | Le bot est le canal principal en Phase 1 ; la Mini App est un complément (D-009) |
| La Mini App n'envoie aucune notification : c'est toujours le bot qui prévient | On garde le bot, même quand la Mini App existe |
| Demande → attente d'une décision du sadran : 2 à 6 minutes, parfois sans réponse | Notre claim instantané est un avantage (D-013) |
| Les détails de la course arrivent à la main, par WhatsApp, envoyés par le sadran | Chez nous, le backend envoie les détails au gagnant automatiquement, dans Telegram |
| Le message du bot n'est pas mis à jour quand la course est prise : des chauffeurs cliquent sur des courses parties et reçoivent une erreur | Notre bot marque « נלקחה » sur les messages des autres à la réception de R ; c'est obligatoire |
| 120 à 240 messages par heure pour un chauffeur | À notre volume, c'est gérable ; mais le filtrage par disponibilité deviendra important quand le volume grandira |
| Inscription avec validation humaine | Notre pré-inscription par le sadran est le bon modèle |
| Disponibilité par trajets et rayons, avec expiration | Phase 1 : un interrupteur avec fin optionnelle ; zones et rayon plus tard |
| Le bot demande « dans combien de minutes es-tu à l'adresse ? », avec un lien Waze | Idée à garder : question Q10 |
| Le chauffeur clôt lui-même (« client annulé », « clôture du paiement ») | Argument pour Q3, plus tard |
| Les chauffeurs haredim utilisent visiblement sa Mini App | Encourageant pour les téléphones filtrés, mais pas une preuve : le test TMA-02 reste nécessaire |

## Questions ouvertes

Liste unique des questions sur la Mini App. Tant qu'une question n'est pas tranchée, on applique la proposition pour la Phase 1. Une réponse devient une entrée de [`DECISIONS.md`](DECISIONS.md), puis la ligne est retirée d'ici.

| # | Question | Qui tranche | Proposition Phase 1 | Bloque |
| --- | --- | --- | --- | --- |
| Q1 | La Mini App entre-t-elle vraiment en Phase 1, vu la charge d'Oren et d'Ilan ? | Tous | Oui, après le bouton du bot ; la porte G1 n'en dépend pas (D-009) | Planning de la Phase 1 |
| Q2 | Notifier en message privé plutôt que dans un groupe ? Relancer à 60 s seulement les chauffeurs disponibles, ou tous les inscrits ? | Ilan, Papa | Message privé ; relance aux chauffeurs disponibles (D-014) | TMA-11 |
| Q3 | Le chauffeur clôt-il lui-même sa course (terminée, client absent) depuis la Mini App ? | Oren, avec Yossef | Non : le sadran clôt, car c'est ce qui écrit au grand livre. Le concurrent laisse le chauffeur clore ; à rouvrir après G1 | — |
| Q4 | Le chauffeur peut-il se désister ? Faut-il une transition `claimed → posted` (course remise en diffusion par le sadran) ? | Oren, Ilan | Non : il appelle le sadran, qui annule et recrée la course | — |
| Q5 | Un chauffeur peut-il avoir plusieurs courses `claimed` en même temps (courses réservées à l'avance) ? | Papa, Oren | Pas de limite | — |
| Q6 | Il manque une table des sadranim et un contrat de gestion des chauffeurs (pré-inscription, blocage avec révocation des sessions, déliaison Telegram) | Oren, avec Ilan | À écrire avant TMA-05 | TMA-05 |
| Q7 | Quel domaine pour héberger la Mini App, et faut-il le faire autoriser par les fournisseurs de filtres des téléphones casher ? | Ilan, Papa | Domaine stable dédié ; réponse attendue du test TMA-02 | TMA-02, TMA-13 |
| Q8 | Le message privé du bot au gagnant (téléphone du client) reste dans l'historique Telegram : faut-il le masquer après la clôture ? | Eitan (avocat, amendement 13) | Le garder en Phase 1, en attendant l'avis | Pilote |
| Q9 | Formulation des boutons et ton (masculin ou neutre) | Papa, avec les 10 testeurs | Textes du tableau « Textes clés » | — |
| Q10 | Demander au gagnant, juste après le claim, dans combien de minutes il sera à l'adresse (avec un lien Waze) ? Utile au sadran, et plus tard pour informer le client | Ilan, Papa | Pas en Phase 1 ; si oui, une question **après** le claim, pour ne jamais le ralentir | — |
