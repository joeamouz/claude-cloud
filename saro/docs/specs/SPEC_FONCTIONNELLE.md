# SARO Manager — Spécification fonctionnelle v2

> Document de référence métier. Il complète `CLAUDE.md`, qui reste court et renvoie ici.
> Règle d'or : **tout est paramétrable** (types, templates, zones, rôles, seuils d'alerte, menus).
> Les éléments marqués **[À CONFIRMER]** sont des hypothèses à valider avec le métier avant de coder.

---

## 0. Glossaire métier

| Terme | Définition |
|---|---|
| **Église / Site** | Entité racine. Prévoir le multi-sites dès le modèle, même si un seul site est actif au départ. |
| **Zone** | Regroupement géographique d'une ou plusieurs communes. Ex. « Yopougon » (1 commune) ou « Abidjan Sud » (Treichville, Marcory, Port-Bouët, Koumassi). Le découpage **évolue avec la croissance** de l'église. |
| **Tribu** | Groupe de membres conduit par un **patriarche** (qui est un berger, voir ci-dessous). Il se réunit 10 minutes chaque dimanche après le culte. Les tribus portent aussi le **programme de cleaning**. |
| **Berger** | Titre commun à **tous les responsables de premier niveau** : responsable de commission, responsable de zone, patriarche (responsable de tribu), assistant du pasteur. Le périmètre d'un berger est **l'unité qu'il dirige** (sa commission, sa zone, sa tribu…). Une même personne peut diriger plusieurs unités : son périmètre est alors l'union de ces unités. Les bergers ont leur propre réunion après les écoles du dimanche. |
| **Commission** | Groupe de service : louange, art & chorégraphie, gestion des cultes, protocole, etc. Elle organise des répétitions, des prières et d'autres activités en semaine. |
| **Gestion des cultes** | Commission chargée de la **gestion du culte dans tous les sens du terme** : organisation, programme et déroulé, timing, coordination des intervenants et des commissions de service (louange, protocole, technique…), logistique, conduite du culte en direct, diffusion du programme et bilan. |
| **Protocole / Assistante du pasteur** | Gère les réceptions pastorales du dimanche (statistiques uniquement). |
| **École** | Formation en présentiel le dimanche après-midi (école de disciples, de baptême, de leadership…). Elle fonctionne par année académique. |
| **PDA** | Plan d'action : action avec un porteur, une échéance et un statut, suivie **jusqu'à sa clôture**. |
| **CR** | Compte rendu de réunion. |
| **Bilan** | Formulaire rempli après une activité. Sa structure est définie par un **template paramétrable**. |

---

## 1. Organisation et périmètres (socle de tout le reste)

### 1.1 Arbre organisationnel paramétrable
- Table générique `org_unit` (id, type, nom, parent_id, responsable_id, actif, valid_from, valid_to).
- Les types d'unité sont paramétrables : site, zone, tribu, commission, école, classe…
- Un membre peut appartenir à plusieurs unités : une tribu, une ou plusieurs commissions, des écoles.
- **Historisation obligatoire** : quand un membre change de tribu, ou quand une zone est redécoupée, l'historique est conservé. Les statistiques passées restent alors justes (pas de réécriture du passé).

### 1.2 Rôles et périmètres
Le contrôle d'accès combine **rôle × périmètre** (RBAC + ABAC) :

| Rôle | Périmètre de données par défaut |
|---|---|
| Pasteur principal / administrateur | Toute l'église |
| Pasteurs | Toute l'église, ou les zones assignées |
| **Berger** (titre commun des responsables de premier niveau) | **L'unité qu'il dirige**, ou l'union de ses unités s'il en dirige plusieurs : |
| ↳ Responsable de zone | Sa zone : toutes les tribus et tous les membres de la zone |
| ↳ Patriarche | Sa tribu |
| ↳ Responsable de commission | Sa commission |
| ↳ Assistant du pasteur | Le périmètre délégué par le pasteur (paramétrable) |
| Directeur d'école / enseignant | Son école / ses classes |
| Gestion des cultes | Cultes, programmes, bilans de culte |
| Protocole / assistante pasteur | Réceptions pastorales (statistiques), réunions qu'elle organise |
| Membre | Ses propres données, son parcours, ses inscriptions |

- Les rôles et les droits (lecture / création / modification / validation / export) sont paramétrables par module.
- Le rôle « berger » n'est pas un rôle isolé : c'est un **titre** porté par toute personne qui est responsable d'une unité (`org_unit.responsable_id`). Son périmètre se **déduit automatiquement** des unités qu'il dirige. Ainsi, nommer un nouveau responsable de tribu lui donne immédiatement les bons accès, sans configuration manuelle.
- Le **périmètre est appliqué côté serveur** sur TOUTES les requêtes : API, exports, carte, tableau de bord **et assistant IA**. Utiliser la Row Level Security si la base le permet.

### 1.3 Fiche membre et carte de membre (ID CARD)

**Fiche membre** : identité, photo, date de naissance, sexe, situation familiale, profession, contacts, adresse (commune / quartier), date d'arrivée, date de conversion et de baptême, tribu, zone, commission(s), fonctions, écoles suivies, parcours de croissance, historique des présences.

**Bouton « Générer la carte de membre »** sur la fiche → carte d'identité du membre, au format carte bancaire (CR80 : 85,6 × 54 mm), recto verso.

- **Recto** :
  - logo et nom de l'église, aux couleurs de la charte,
  - **photo** du membre,
  - nom, prénom,
  - **numéro de membre** unique (matricule, ex. `SARO-2026-00123`),
  - tribu, zone,
  - fonction ou titre (berger, serviteur de la commission X, membre…),
  - **QR code**.
- **Verso** :
  - profession, âge (ou date de naissance, au choix),
  - **ancienneté** (« Membre depuis 2019 · 7 ans »), date de baptême,
  - commission(s), écoles validées,
  - contact d'urgence (nom + téléphone),
  - date d'émission et date de validité,
  - signature du pasteur (image), mention « Cette carte reste la propriété de l'église ».

**Fonctionnement**
- **Modèle de carte paramétrable** : choix des champs affichés au recto et au verso, couleurs, logo, durée de validité. Un administrateur peut créer plusieurs modèles (membre, serviteur, berger, visiteur).
- **QR code sécurisé** : il contient un jeton signé, pas les données personnelles en clair. Une fois scanné dans l'application, il sert à :
  - **vérifier** la carte (valide, expirée ou révoquée, avec photo pour contrôle visuel),
  - **pointer la présence** au culte, en tribu ou en école en un geste (check-in).
- **Sorties** :
  - PDF imprimable à l'unité,
  - **impression en lot** (planche A4 de 8 ou 10 cartes, par exemple pour toute une tribu),
  - **carte numérique** dans l'application du membre (consultable hors ligne).
- **Cycle de vie** : émission, renouvellement à l'expiration, **révocation** (perte, départ), réédition avec un nouveau QR code. Historique des cartes émises.
- **Contrôles** :
  - génération impossible sans photo ni champs obligatoires (liste des champs manquants affichée),
  - recadrage automatique de la photo.
- **Confidentialité** : les données sensibles (adresse, téléphone personnel) ne sont jamais imprimées par défaut.

---

## 2. Carte interactive (Côte d'Ivoire + monde)

### 2.1 Décision technique
- **Carte du monde : OUI, c'est faisable.**
  - Web : **MapLibre GL JS**.
  - Android : **@maplibre/maplibre-react-native**.
  - Open source, tuiles vectorielles, fluide, animations de caméra (`flyTo`), polygones, clusters, cartes de chaleur.
- Fonds de carte sans clé obligatoire : OpenFreeMap, ou PMTiles auto-hébergés (Protomaps). Alternative : MapTiler (offre gratuite).
- Limites administratives à **vérifier et importer** :
  - Côte d'Ivoire : geoBoundaries / HDX COD-AB / OpenStreetMap, au niveau commune.
  - Monde : Natural Earth (pays).
- Abidjan : vérifier la couverture des 10 communes historiques + Anyama, Bingerville, Songon.
- **Performance web** : la librairie carte est chargée en différé, uniquement sur l'écran Carte (lazy import). Elle ne doit pas peser sur le premier chargement (objectif Lighthouse).

### 2.2 Zones dynamiques
- Tables :
  - `commune` (référentiel + géométrie GeoJSON),
  - `zone`,
  - `zone_commune` (zone_id, commune_id, valid_from, valid_to).
- La géométrie d'une zone est **calculée** par union des communes (PostGIS `ST_Union` ou turf.js), puis mise en cache.
- Écran d'administration des zones :
  - créer, fusionner ou scinder une zone,
  - glisser une commune d'une zone vers une autre directement sur la carte,
  - prévisualiser l'impact (nombre de membres et de tribus touchés),
  - choisir une date d'effet.
- Diaspora (France, etc.) : les zones peuvent être de type « Pays » ou « Ville » hors Côte d'Ivoire. Exemple : « France – Île-de-France ».

### 2.3 Navigation « augmentée »
- **Zoom progressif (drill-down)** : Monde → Pays → Côte d'Ivoire → Zone → Commune. Animation de caméra à chaque niveau.
- **Clic sur une zone ou une commune** → panneau latéral animé avec :
  - membres actifs, évolution sur 3, 6 et 12 mois, répartition hommes / femmes / enfants,
  - tribus et patriarches, bergers, taux de présence au culte et en tribu,
  - nouveaux venus, membres à suivre (absences),
  - prochaines activités, PDA ouverts,
  - bouton « Analyser cette zone avec l'IA ».
- **Couches activables** :
  - choroplèthe (membres, croissance, taux de présence),
  - clusters de membres,
  - carte de chaleur des nouveaux venus,
  - lieux d'activités,
  - limites des zones.
- **Curseur temporel** : rejouer la croissance de l'église mois par mois (animation).
- **Confidentialité** : aucune adresse individuelle n'est visible, sauf pour les rôles autorisés. Par défaut, les données sont agrégées à la commune ou au quartier. Géocodage au quartier, pas à la porte.

---

## 3. Programme d'activités et alertes

### 3.1 Modèle
- **Programme** : annuel, trimestriel ou mensuel. Il est rattaché à **n'importe quelle unité de l'organisation** : église, commission, tribu, zone, école… (via `org_unit`). Il contient des **activités**.
  - Les programmes des unités se **consolident** dans le programme de l'église : le pasteur voit tout, chaque berger voit et gère le programme de son unité.
  - Le responsable de l'unité est propriétaire de son programme. Les activités d'une unité peuvent être soumises à validation de l'échelon supérieur (paramétrable).
- **Activité** :
  - type (paramétrable), titre, description,
  - date et heure de début et de fin, récurrence (RRULE),
  - lieu, commission organisatrice, responsable, intervenants,
  - budget prévisionnel et réel,
  - statut, template de bilan associé.
- **Statuts** : Brouillon → Planifiée → Confirmée → En préparation → En cours → Réalisée → Bilan fait. Statuts de sortie : Reportée, Annulée (avec motif obligatoire).
- **Checklist de préparation par type d'activité** (paramétrable), avec des jalons relatifs. Exemple pour une « Convention » :
  - J-60 : réserver la salle
  - J-30 : lancer la communication
  - J-14 : confirmer les intervenants
  - J-7 : répétition générale
  - J+3 : remettre le bilan
- À la création d'une activité, les jalons sont **générés automatiquement en tâches PDA** avec leur porteur et leur échéance.

### 3.2 Alertes (seuils paramétrables par type d'activité)

| Alerte | Déclencheur par défaut | Destinataires |
|---|---|---|
| Échéance proche | J-30, J-7, J-1 avant l'activité | Responsable, commission |
| Tâche bientôt due | J-2 avant l'échéance | Porteur |
| Retard | Échéance dépassée et tâche non terminée | Porteur, puis **escalade** au responsable à J+2, puis au pasteur à J+7 |
| Bilan manquant | Activité passée sans bilan après N jours | Responsable |
| Conflit | Même salle ou même responsable sur le même créneau | Organisateurs |
| Activité non confirmée | J-14 et toujours au statut « Planifiée » | Responsable |

- Canaux : bandeau INFOS, centre de notifications, push, e-mail, WhatsApp **[À CONFIRMER : fournisseur]**.
- Les alertes sont calculées par un **job planifié côté serveur** (cron), pas par le client.

### 3.3 Vues
- Calendrier mois / semaine / agenda, filtrable par commission, zone et type.
- Vue **Gantt / chronologie** du programme, avec les jalons.
- Vue **« Ma semaine »** par utilisateur : mes activités, mes tâches, mes bilans à remplir.
- Export PDF et iCal. Abonnement au calendrier (lien ICS).

---

## 4. Plans d'action (PDA) — module unique et transverse

### 4.1 Principe
**Une seule table `action`**, quelle que soit l'origine de l'action. Un lien polymorphe (`source_type`, `source_id`) indique d'où elle vient :
- un programme ou une activité,
- un point de réunion,
- un bilan,
- le cleaning,
- une saisie libre,
- une suggestion de l'IA.

Tous les PDA de l'église sont ainsi visibles dans un seul tableau, avec des filtres par origine.

### 4.2 Champs
- Titre, description.
- **Porteur principal** (obligatoire) + contributeurs.
- Échéance, priorité (basse / normale / haute / critique).
- Statut : À faire → En cours → Bloqué (motif) → Terminé → **Clôturé** (validé par le propriétaire de la source). Autre statut possible : Annulé (motif).
- % d'avancement.
- Fil de commentaires, pièces jointes, historique complet des changements (journal d'audit).
- Récurrence (pour les actions répétitives comme le cleaning).
- **Report d'échéance tracé** : ancienne date, nouvelle date, motif, nombre de reports. C'est un indicateur clé de pilotage.

### 4.3 Règles
- Une action n'est **jamais clôturée automatiquement**. Elle reste suivie jusqu'à sa clôture explicite.
- Les actions ouvertes d'une réunion sont **reprises automatiquement** à l'ordre du jour de la réunion suivante du même type (« Revue des PDA »).
- Vues :
  - Kanban,
  - liste (filtres par porteur, origine, statut, retard),
  - « Mes actions »,
  - tableau de performance par porteur : taux de réalisation dans les délais, retards, reports.

### 4.4 Programme de cleaning (cas particulier de PDA récurrent)
- Paramétrage :
  - zones de nettoyage (salle principale, sanitaires, parvis…),
  - fréquence,
  - tribus participantes.
- **Générateur de planning en rotation** entre les tribus sur N semaines. Il est modifiable à la main et permet d'échanger deux créneaux.
- Chaque créneau génère une action pour le patriarche de la tribu.
- Validation par le responsable du site : fait / partiel / non fait, avec une photo optionnelle et une note de qualité.
- Statistiques : taux de réalisation par tribu, alertes en cas de manquement.

---

## 5. Réunions (tous types)

### 5.1 Types de réunion paramétrables
Exemples : réunion des bergers/responsables, réunion de commission, conseil pastoral, réunion de tribu, comité d'organisation d'un événement.

Pour chaque type, on paramètre :
- les participants par défaut (par rôle ou par unité),
- la périodicité,
- le **modèle d'ordre du jour**,
- le **modèle de CR**,
- la liste de diffusion par défaut,
- le quorum.

### 5.2 Cycle de vie d'une réunion
1. **Planifiée** : date, lieu ou lien visio. L'ordre du jour est prérempli avec :
   - le modèle du type de réunion,
   - les **points en suspens** de la réunion précédente,
   - la **revue des PDA ouverts**.
2. **Convoquée** : invitation par notification, e-mail ou WhatsApp, avec confirmation de présence.
3. **Tenue** :
   - présences par pointage rapide,
   - saisie des notes, en texte libre ou point par point.
   - Mode hors ligne obligatoire : la connexion peut manquer sur place.
4. **CR brouillon généré par l'IA** à partir des notes, au format :
   - en-tête (type, date, lieu, président de séance, secrétaire, présents / absents / excusés),
   - résumé exécutif en 3 à 5 lignes,
   - points traités : discussion → **décision**,
   - **PDA proposés**, extraits des notes : action, porteur suggéré, échéance suggérée. Ils sont à valider un par un.
   - points reportés, prochaine réunion.
5. **Relecture et modification** par l'assistante. Le CR est **versionné**.
6. **Validation** par le président de séance ou le pasteur. Une fois validé, le CR est figé (une nouvelle version reste possible, tracée).
7. **Diffusion** :
   - PDF à la charte de l'église, envoyé par e-mail à une **liste de diffusion paramétrable**,
   - lien sécurisé dans l'application,
   - option WhatsApp.
8. **Suivi** : les PDA validés entrent dans le module PDA et les points non clos remontent à la réunion suivante.

### 5.3 « Points à traiter »
- Un point d'ordre du jour a un statut : Ouvert / Traité / Reporté / Clôturé.
- Il reste visible et suivi de réunion en réunion **tant qu'il n'est pas clôturé**, avec son historique (dans quelles réunions il a été discuté).

---

## 6. Bilans paramétrables (moteur de formulaires)

- **Constructeur de templates** (form builder). Types de champs :
  - nombre, texte court ou long, choix unique ou multiple, date, heure,
  - case à cocher, note de 1 à 5, photo / pièce jointe,
  - sélection de membres, compteurs hommes / femmes / enfants.
- Actions possibles : créer, modifier, dupliquer, **désactiver** un template (jamais supprimer un template déjà utilisé).
- **Versionnage** : un bilan garde la référence de la version de template utilisée, et les anciens bilans restent lisibles.
- Un template s'associe à un type d'activité, une commission, une tribu, un culte ou une école.
- **Stockage analytique** :
  - les réponses sont stockées en JSON (`answers`),
  - **les indicateurs clés** (heures, effectifs H/F/E, visiteurs…) sont aussi copiés dans des **colonnes typées** ou une table de faits (`fact_attendance`). Ainsi, SQL, tableaux de bord et IA peuvent les interroger de façon fiable.
- Champs calculés (total = H + F + E) et règles de validation (heure de fin > heure de début, total cohérent).
- Rappel automatique si un bilan attendu n'est pas rempli.

---

## 7. Semaine

### 7.1 Activités des commissions
- Répétitions, prières et autres activités au choix de chaque commission, créées dans le **programme d'activités** avec le type adapté.
- Bilan rempli dans le groupe avec le template de la commission.
- Présences nominatives des membres de la commission → taux d'engagement de chaque serviteur.

### 7.2 Cleaning par tribu
Voir §4.4.

### 7.3 Diffusion du programme du dimanche
- La responsable de la gestion des cultes compose le **déroulé du culte** à partir d'un modèle :
  - séquences (accueil, louange, prière, annonces, offrande, prédication…),
  - heure prévue et durée de chaque séquence,
  - intervenant de chaque séquence,
  - besoins techniques (son, vidéo, chorégraphie).
- **Diffusion** en un clic aux responsables (pasteurs, assistants, bergers, serviteurs concernés) : notification + PDF + lien.
- Accusé de lecture.
- Chaque intervenant voit **ses** passages, mis en évidence.
- Modifications de dernière minute → notification « Programme mis à jour » avec les changements surlignés.

---

## 8. Dimanche

### 8.1 Conduite du culte en direct (« conducteur »)
- Mode plein écran pour la gestion des cultes : séquence en cours, chronomètre, séquence suivante, **écart entre le prévu et le réel** en temps réel.
- Un appui marque le début et la fin de chaque séquence → les heures réelles sont enregistrées automatiquement dans le bilan.

### 8.2 Bilan du culte (gestion des cultes)
Champs proposés (template par défaut, modifiable) :
- **Identification** : date, site, numéro du culte (1er / 2e…), thème, texte biblique, prédicateur.
- **Horaires** : début et fin prévus / réels (préremplis par le conducteur), durée, retard au démarrage.
- **Effectifs** :
  - hommes, femmes, enfants (garçons / filles en option), adolescents (en option),
  - total calculé automatiquement,
  - **nouveaux venus / visiteurs**, nouveaux convertis, en ligne (si diffusion live).
  - **Méthode de comptage** : comptage manuel, check-in ou estimation (indice de fiabilité).
- **Service** : nombre de serviteurs mobilisés par commission, absences de serviteurs.
- **Qualité** : note de 1 à 5 par séquence importante, points forts, points à améliorer.
- **Incidents** : technique, sécurité, santé, logistique (catégorie + description), avec possibilité de **créer un PDA** directement.
- **Contexte** : événement exceptionnel (fête, pluie forte, jour férié). C'est utile pour expliquer les variations dans les analyses.
- **Commentaire libre**, puis **résumé généré par l'IA** à partir des chiffres et du commentaire.
- Stocké dans `service_report` + `fact_attendance` (voir §6).

### 8.3 Bilan de tribu (patriarche, 10 minutes après le culte)
- **Recommandé : présence nominative rapide.** La liste des membres de la tribu s'affiche et on coche les présents d'un appui (fonctionne hors ligne).
  - Les effectifs H/F/E sont **calculés automatiquement** à partir de la fiche des membres.
  - Les **absents sont connus**, ce qui permet les alertes de suivi.
  - Saisie manuelle des totaux possible en secours.
- Autres champs :
  - heure de début et de fin,
  - nouveaux venus intégrés à la tribu (création rapide de fiche),
  - **sujets de prière / besoins** (catégories : santé, emploi, famille, finances, spirituel),
  - **événements de vie** : naissance, mariage, deuil, maladie, voyage → alerte pastorale et action de suivi proposée,
  - commentaire.
- Règles automatiques (paramétrables) :
  - membre absent **3 dimanches consécutifs** → alerte au patriarche et au berger + action « Prendre des nouvelles »,
  - nouveau venu présent 2 fois → proposé à l'intégration.

### 8.4 Réceptions pastorales (protocole / assistante)
Ces séances sont **privées** : **aucun nom ni contenu** n'est enregistré par défaut. Seulement des statistiques :
- nombre de personnes reçues, hommes / femmes, tranche d'âge (enfant, jeune, adulte, senior),
- statut : membre / visiteur / nouveau converti,
- **catégorie de motif**, non nominative et facultative : spirituel, famille / couple, santé, emploi / finances, délivrance / prière, orientation, autre,
- reçu seul / en couple / en famille,
- durée de chaque entretien, temps d'attente,
- **personnes non reçues faute de temps** (indicateur de charge pastorale) et reportées.
- **File d'attente numérique** (en option) : le protocole enregistre les arrivées dans l'ordre, appelle la personne suivante et les durées sont mesurées automatiquement.
- Accès réservé au pasteur, à l'assistante et au protocole. Les statistiques agrégées sont visibles par les rôles autorisés.

### 8.5 Écoles et cours (après-midi) — « épate-moi »
**Paramétrage**
- Écoles (disciples, baptême, leadership, école biblique…), niveaux, promotions et classes.
- Année académique (dates, trimestres ou modules), salles.
- Enseignants, programme des cours (modules et séances).

**Inscription**
- Inscription manuelle, ou via une demande du membre dans l'application, validée par le directeur.
- **Prérequis** paramétrables. Exemple : avoir validé « Disciples 1 » pour entrer en « Disciples 2 ».

**Présences**
- Appel en 30 secondes sur le téléphone de l'enseignant : tous présents par défaut, on décoche les absents.
- **Ou** auto-pointage de l'élève en scannant le **QR code de la séance**.
- Mode hors ligne.
- Justification d'absence (motif + pièce) et séance de **rattrapage**.

**Règles de validation** (paramétrables par école)
- Taux de présence minimum (ex. 75 %).
- En option : notes d'évaluation, devoirs, projet.
- Calcul automatique du statut : En bonne voie / **À risque** / Non validable / Validé.

**Alertes**
- Élève sous le seuil à mi-parcours → alerte à l'élève, à l'enseignant et au berger / patriarche.
- Enseignant qui n'a pas fait l'appel.

**Fin d'année**
- Délibération assistée.
- **Attestation / certificat PDF** avec un QR code de vérification.
- Bulletin individuel.

**Parcours de croissance**
Les écoles alimentent le **parcours du membre** (voir §9.2). Chaque validation fait avancer le membre dans son parcours.

**Tableau de bord école**
- Inscrits, taux de présence par séance et par classe, élèves à risque, taux de réussite, comparaison entre promotions.

### 8.6 Réunion des bergers / responsables
Réunion de type « Réunion des bergers ». Elle suit le cycle complet décrit au §5.2, en particulier :
- notes saisies par l'assistante,
- **CR percutant généré par l'IA**, modifiable puis validé,
- **PDF diffusé à la liste de diffusion par e-mail**,
- **PDA proposés à partir des notes**, validés puis suivis jusqu'à leur clôture.

Ordre du jour suggéré automatiquement avec les données de la semaine : chiffres du culte, absences à suivre, alertes, PDA en retard.

---

## 9. Tableau de bord orienté croissance et suivi des membres

### 9.1 Indicateurs clés (KPI)
Indicateurs reconnus du pilotage d'église en croissance. Chacun est affiché avec sa tendance et sa comparaison à N-1 :

| Famille | KPI |
|---|---|
| **Fréquentation** | Présence moyenne au culte (moyenne mobile sur 4 semaines), répartition H/F/E, taux de remplissage de la salle |
| **Accueil** | Nouveaux venus par semaine ou par mois, **taux de retour** (2e et 3e visite), délai de premier contact |
| **Intégration** | % des membres actifs rattachés à une tribu, % présents en tribu le dimanche |
| **Engagement** | **% de membres qui servent** dans une commission, nombre de serviteurs par commission |
| **Formation** | Inscrits en école, taux de présence, taux de validation |
| **Croissance** | Croissance nette (entrées − sorties / inactifs), baptêmes, nouveaux convertis |
| **Rétention** | Membres « à risque » (absents 3 semaines ou plus), taux de réactivation après suivi |
| **Encadrement** | Ratio membres / patriarche et membres / berger (alerte au-delà d'un seuil, signe qu'il faut ouvrir une nouvelle tribu) |
| **Exécution** | Activités réalisées vs planifiées, **PDA dans les délais**, PDA en retard par porteur, bilans remis |
| **Pastoral** | Réceptions par dimanche, personnes non reçues |
| **Géographie** | Membres et croissance par zone (mini-carte cliquable) |

### 9.2 Visualisations phares
- **Entonnoir du parcours du membre** : Visiteur → Revenu → Intégré en tribu → Converti / baptisé → Formé (école) → Serviteur → Leader. Pour chaque étape : volumes, taux de passage et durée moyenne. C'est **le** graphique d'une église axée sur la croissance.
- Courbe de fréquentation sur 52 semaines (H/F/E empilés, moyenne mobile, annotations des événements).
- **Carte de chaleur des présences en tribu** (tribus × semaines).
- Carte des zones (choroplèthe de la croissance).
- Liste « **Membres à suivre cette semaine** » (absences, événements de vie, nouveaux venus sans contact), avec une action en un clic.
- Bloc « **À venir sur 14 jours** » (activités, échéances) et « **Alertes** ».
- **Synthèse IA de la semaine**, en 5 points, en haut du tableau de bord.

### 9.3 Tableaux de bord par rôle
Pasteur (global), gestion des cultes, bergers (responsable de zone, patriarche, responsable de commission, assistant du pasteur), directeur d'école.

Chaque tableau de bord est limité au **périmètre** du rôle et ses widgets sont paramétrables (affichage, ordre).

---

## 10. Assistant IA et module de synthèse

### 10.1 Objectif
Une intelligence interne capable de produire des **synthèses et analyses percutantes** sur l'ensemble de l'application, **strictement limitées au périmètre** de l'utilisateur :
- le pasteur → toute l'église,
- un berger → l'unité qu'il dirige : un responsable de zone voit sa zone, un responsable de commission sa commission,
- un patriarche → sa tribu,
- un responsable de commission → sa commission,
- un directeur d'école → son école.

### 10.2 Architecture
- **Chiffres** : l'IA appelle des **outils (tool use)** qui exécutent des requêtes SQL paramétrées. Le filtre de périmètre est **injecté côté serveur** à partir de l'identité de l'utilisateur, **jamais à partir du texte de la question**. L'IA n'invente aucun chiffre.
- **Textes** : RAG (pgvector, recherche hybride) sur les CR, bilans, commentaires, documents et le glossaire. Chaque morceau indexé porte des métadonnées de périmètre (`org_unit_id`, niveau de confidentialité). La recherche est filtrée **avant** la récupération.
- **Exclusions** : les réceptions pastorales nominatives (il n'y en a pas par défaut) et les données marquées confidentielles ne sont jamais indexées.
- LLM : API Claude (Anthropic). Réponses en streaming, **sources citées et cliquables**.
- Journal d'audit des questions posées (qui, quand, périmètre).

### 10.3 Fonctions
- **Chat IA** : questions libres, avec des questions suggérées selon le rôle. Exemples :
  - Patriarche : « Qui sont les membres de ma tribu absents depuis 3 semaines ? »
  - Responsable de zone : « Compare la présence des tribus de ma zone ce trimestre. »
  - Responsable de commission : « Quels serviteurs de ma commission ont manqué plus de 2 répétitions ce mois-ci ? »
  - Pasteur : « Quelles zones ont la plus forte croissance et pourquoi ? »
  - Pasteur : « Résume les décisions des 3 dernières réunions des bergers et les PDA en retard. »
- **Synthèses automatiques** :
  - synthèse hebdomadaire par rôle (lundi matin),
  - synthèse du culte,
  - synthèse mensuelle de l'église (PDF).
- **Générateurs** :
  - CR de réunion,
  - extraction des PDA à partir des notes,
  - résumé de bilan,
  - ordre du jour suggéré.
- **Détection proactive** :
  - baisse de fréquentation d'une zone ou d'une tribu,
  - hausse des absences,
  - commission en sous-effectif,
  - porteur de PDA surchargé,
  - tribu à scinder (ratio trop élevé).
  - Ces détections sont poussées en alertes.
- **Graphiques générés dans la réponse** et export PDF.
- **Évaluation** : jeu de questions de test par rôle, **incluant des tests de fuite de périmètre**. Exemple : un patriarche demande des données d'une autre tribu → refus attendu.

---

## 11. Menu optimisé (orienté usage)

Les sections suivent le **rythme de vie de l'église**. Le menu reste **entièrement paramétrable** (`menu_config` : visibilité par rôle, ordre, libellés, icônes, feature flags). Le menu ci-dessous est la configuration **par défaut**.

| Section | Entrées |
|---|---|
| **🏠 ACCUEIL** | Tableau de bord · Ma semaine (mes activités, actions, bilans à remplir) · Alertes |
| **✨ ASSISTANT IA** | Chat IA · Synthèses (hebdomadaire, mensuelle, culte) · Analyses proactives |
| **⛪ DIMANCHE** | Programme du culte · Conducteur (en direct) · Bilan du culte · Bilans de tribu · Réceptions pastorales |
| **👥 COMMUNAUTÉ** | Membres · Parcours de croissance · Nouveaux venus & suivi · Tribus · Zones & carte · Familles |
| **🙌 SERVICE** | Commissions · Activités & bilans des commissions · Planning des serviteurs · Cleaning |
| **🎓 FORMATION** | Écoles · Classes & séances · Présences · Validation & attestations |
| **📅 PILOTAGE** | Programme d'activités (calendrier / Gantt) · Réunions · Plans d'action (PDA) · Comptes rendus · Rapports |
| **📣 COMMUNICATION** | Notifications · Diffusions (listes, e-mail, WhatsApp) · Annonces (bandeau INFOS) |
| **⚙️ ADMINISTRATION** | Organisation (types d'unités, zones) · Rôles & périmètres · Templates de bilans · Types d'activités & de réunions · Menu · Imports · Sauvegardes |
| **❓ AIDE** | Documentation · Glossaire |

Principes :
- **3 clics maximum** pour toute action fréquente du dimanche.
- **Barre d'onglets mobile** par rôle. Exemple pour un patriarche : Accueil · Ma tribu · Bilan · IA · Moi.
- Bouton d'action rapide (+) contextuel : nouveau bilan, nouvelle action, nouveau venu.
- Badges dynamiques sur les entrées : bilans en attente, PDA en retard, alertes.

---

## 12. Exigences transverses
- **Hors ligne d'abord pour le dimanche** : bilans, présences, notes de réunion. File de synchronisation et résolution des conflits.
- **Audit** : qui a créé, modifié ou validé quoi, et quand.
- **RGPD / confidentialité** : consentement des membres, droit d'accès et de rectification, minimisation (réceptions pastorales anonymes).
- **Exports** : PDF à la charte de l'église, Excel.
- **i18n** : français par défaut.
- **Performance web** : voir les objectifs Lighthouse dans `CLAUDE.md`. Carte, graphiques lourds et IA chargés en différé.

---

## 13. Questions ouvertes [À CONFIRMER]
1. ~~Périmètre d'un berger~~ → **tranché** : un berger est tout responsable de premier niveau, son périmètre est l'unité qu'il dirige (§0, §1.2). Reste à confirmer : une tribu appartient-elle toujours à une seule zone ?
2. Nombre approximatif de membres, de tribus, de commissions et d'écoles (pour dimensionner l'application).
3. Un ou plusieurs cultes le dimanche ? Un ou plusieurs sites ?
4. Définition d'« enfant » (âge limite) et besoin de distinguer les adolescents ?
5. Les offrandes sont-elles saisies dans le bilan du culte ou dans un module Finances séparé ?
6. Canal WhatsApp souhaité (WhatsApp Business API, payant) ou simple lien de partage ?
7. Fournisseur d'e-mail pour l'envoi des CR (SMTP, SendGrid, Brevo…) ?
8. Qui valide un CR : le président de séance, le pasteur ou les deux ?
