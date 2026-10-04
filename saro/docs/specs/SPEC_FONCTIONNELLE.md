# SARO Manager — Spécification fonctionnelle v3

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
| **Cellule** | Petite unité de **prière hebdomadaire** par secteur ou quartier, à l'intérieur d'une zone. Une zone peut contenir **plusieurs cellules**. Une cellule est animée par un responsable de cellule (n'importe quel membre désigné) et placée sous la **responsabilité centrale du responsable de zone**. |
| **Commission** | Groupe de service : louange, art & chorégraphie, gestion des cultes, protocole, etc. Elle organise des répétitions, des prières et d'autres activités en semaine. |
| **Unité** | Mot générique pour toute structure de l'église : zone, tribu, cellule, commission, école. Chaque unité a son **espace** dans l'application (§1.5), son **bureau** (§1.4) et ses **ressources**. |
| **Bureau** | Les personnes qui portent les fonctions d'une unité (responsable, adjoint, secrétaire…), choisies **uniquement parmi les membres** (§1.4). |
| **Nouveau venu** | Personne venue à l'église sans en être membre. Suivi par la commission **ADN** en vue de sa **fidélisation et de son intégration** (§14.2). |
| **ADN (Accueil des Nouveaux)** | Commission qui suit les nouveaux venus : appels, relances, orientation vers une tribu, une cellule ou une école. Son responsable est un berger. |
| **Âme / nouveau converti** | Personne qui a pris la décision de suivre Christ. Suivie par la commission **Suivi des âmes** en vue de sa **maturité spirituelle** (§14.3). |
| **Suivi des âmes** | Commission qui accompagne les nouveaux convertis jusqu'à leur maturité spirituelle. Elle est distincte d'ADN (§14.1). |
| **Calling** | Appels téléphoniques de suivi (nouveaux venus, âmes, absents, campagnes) : listes, script, résultat (§14.4). |
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

### 1.1 bis Règles d'appartenance et existant à conserver
- **Hiérarchie** : Pasteur principal > Assistants pasteurs > Bergers (responsables de premier niveau, voir §0) > Serviteurs > Membres. Types supplémentaires : Maman Pasteur, Adjoint de structure/commission. Les types d'acteurs sont **configurables et cumulables**.
- **Appartenances** (décisions du 2026-10-03) :
  - un membre appartient à **au plus une tribu et une zone** à une date donnée, et à **plusieurs commissions** ; contrainte appliquée en base ;
  - une **tribu peut s'étendre sur plusieurs zones** (ses membres habitent des zones différentes) : le périmètre d'un responsable de zone porte sur les **membres de sa zone**, pas sur des tribus entières ;
  - une **commune appartient à une seule zone à une date donnée** ; un redécoupage se fait par date d'effet (historique conservé) ;
  - une **cellule** appartient à une seule zone ; un membre participe à au plus une cellule.
- **Commissions de départ** (corrigeables par l'admin) : accueil, **accueil des nouveaux (ADN)**, **suivi des âmes**, protocole, intercession, louange, média/son, évangélisation, jeunesse, enfants, femmes, hommes. Dans le code actuel, une commission s'appelle « département » (table `department`).
- **Suivi pastoral nominatif** (existant) : notes chiffrées, droits dédiés `pastoral.read/write`, chaque consultation journalisée, jamais accessible à l'administrateur technique. Il coexiste avec les réceptions pastorales anonymes (§8.4).
- **Volumes** (Q2) : moins de 2 000 membres, 50 tribus, 30 commissions, 10 écoles. SQLite suffit ; PostgreSQL reste possible.

### 1.2 Rôles et périmètres
Le contrôle d'accès combine **rôle × périmètre** (RBAC + ABAC) :

| Rôle | Périmètre de données par défaut |
|---|---|
| Pasteur principal / administrateur | Toute l'église |
| Pasteurs | Toute l'église, ou les zones assignées |
| **Berger** (titre commun des responsables de premier niveau) | **L'unité qu'il dirige**, ou l'union de ses unités s'il en dirige plusieurs : |
| ↳ Responsable de zone | Sa zone : toutes les tribus et tous les membres de la zone |
| ↳ Patriarche | Sa tribu |
| ↳ Responsable de cellule | Sa cellule (le responsable de zone voit toutes les cellules de sa zone) |
| ↳ Responsable de commission | Sa commission |
| ↳ Assistant du pasteur | Le périmètre délégué par le pasteur (paramétrable) |
| Directeur d'école / enseignant | Son école / ses classes |
| Gestion des cultes | Cultes, programmes, bilans de culte |
| Responsable ADN (berger) | Tous les nouveaux venus de l'église et leurs statistiques ; répartit les appels (§14.2) |
| Membre de la commission ADN | Uniquement les nouveaux venus qui lui sont assignés |
| Responsable du suivi des âmes (berger) | Toutes les âmes en suivi et leurs statistiques ; assigne les accompagnateurs (§14.3) |
| Accompagnateur (membre de la commission Suivi des âmes) | Uniquement les âmes qui lui sont assignées |
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
  - **numéro de membre** unique (matricule `SARO-<année d'arrivée>-<numéro>`, ex. `SARO-2019-00123` ; préfixe et format paramétrables ; attribué une fois, jamais réutilisé),
  - tribu, zone,
  - fonction ou titre (berger, serviteur de la commission X, membre…),
  - **QR code**.
- **Verso** :
  - profession, âge (ou date de naissance, au choix),
  - **ancienneté** (« Membre depuis 2019 · 7 ans »), date de baptême,
  - commission(s), écoles validées,
  - contact d'urgence (nom + téléphone),
  - date d'émission (**pas de date d'expiration** : décision du 2026-10-03 ; une durée de validité reste paramétrable par modèle si besoin plus tard),
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
- **Cycle de vie** : émission, **révocation** (perte, départ), réédition avec un nouveau QR code. Historique des cartes émises. Pas d'expiration par défaut.
- **Badges d'activité** : la carte et son QR code servent à produire rapidement des **badges** pour les activités (convention, séminaire…) : planche de badges pour une liste d'inscrits, avec le nom de l'activité. Chaque membre a **son QR code personnel** (le même jeton signé que la carte).
- **Signature du pasteur** : image facultative, chargée par l'administrateur dans le modèle de carte.
- **Contrôles** :
  - génération impossible sans photo ni champs obligatoires (liste des champs manquants affichée),
  - recadrage automatique de la photo.
- **Confidentialité** : les données sensibles (adresse, téléphone personnel) ne sont jamais imprimées par défaut.

### 1.4 Bureau des unités et règle « tout vient de Membres »

**Bureau d'une unité**
- Les **fonctions** sont paramétrables par type d'unité (responsable, adjoint, secrétaire, chargé de la communication, chargé de la logistique…). Aucune fonction financière n'est proposée par défaut (décision « aucune finance »).
- Chaque fonction a un **mandat daté** (début, fin), historisé : on sait qui occupait quelle fonction à une date donnée.
- Les fonctions **responsable** et **adjoint** donnent automatiquement les droits de l'espace d'unité (§1.5), comme pour les bergers (§1.2). **Par défaut, un adjoint a les mêmes droits que le responsable** (décision du 2026-10-04) ; cela reste paramétrable pour pouvoir réduire les droits des adjoints plus tard.
- Chaque unité **déclare elle-même son bureau** dans son espace (onglet Bureau).

**Règle transverse : toute personne désignée est un membre**
- Toute désignation d'une personne dans l'application (bureau, porteur de PDA, intervenant au culte, accompagnateur, appelant, enseignant, délégué…) se fait **en sélectionnant une fiche du registre des membres**. **Aucune saisie libre de nom.** Cette règle est appliquée **en base de données**, pas seulement dans l'écran.
- Pour qu'une personne puisse être désignée à une fonction, elle doit avoir le statut **Membre**.
- **Registre unique des personnes** avec un **statut** : *Nouveau venu* → *Membre* → *Inactif* → *Parti*. Les nouveaux venus et les âmes (§14) sont dans le même registre que les membres : l'intégration d'un nouveau venu **change son statut** sans recréer sa fiche (le matricule SARO est alors attribué).
- Être « âme en suivi » n'est pas un statut mais un **suivi parallèle** (§14.3) : un membre déjà ancien peut aussi y entrer.
- À vérifier dans le code existant : la table des nouveaux venus actuelle est **réutilisée**, pas dupliquée (Claude Code propose la migration réversible).

### 1.5 Espace d'unité, délégations et ressources

Chaque unité (tribu, cellule, commission, zone, école) a **le même espace**, avec les mêmes onglets. Ce n'est pas un module de plus.

| Onglet | Contenu |
|---|---|
| **Aperçu** | Chiffres de l'unité : effectif, présence, activités à venir, PDA en retard |
| **Bureau** | Fonctions et titulaires (§1.4) |
| **Membres** | Liste, présences, engagement |
| **Activités** | Le programme de l'unité (§3) : répétitions, prières, sorties… avec leurs bilans |
| **Ressources** | L'archive de l'unité (voir ci-dessous) |
| **Réunions & PDA** | Réunions de l'unité, comptes rendus et actions |

Les écoles ajoutent les onglets Classes, Présences et Attestations (§8.5).

**Qui gère l'espace**
- Le **responsable** et les **adjoints** (mêmes droits par défaut) gèrent tout ; chacun voit **son** espace.
- Le responsable peut **déléguer** à un membre des droits précis : gérer les activités, saisir les bilans, gérer les ressources, faire l'appel. La délégation a une **date de fin facultative**, est **révocable** et est **tracée** dans le journal d'audit. Le délégué est forcément un membre (§1.4).

**Ressources de l'unité (archivage)**
- **Documents** (PDF, images, documents bureautiques, partitions, listes) **stockés sur le serveur**, limités à **10 Mo par fichier** (paramétrable). Types de fichiers autorisés paramétrables.
- **Audio et vidéo : liens uniquement** (YouTube, Drive…), pour ne pas surcharger le serveur.
- **Quota d'espace par unité**, paramétrable, avec un message clair quand il est atteint.
- Chaque ressource a un titre, une description, des étiquettes, une date et l'auteur ; recherche dans l'unité ; historique de modification.
- Les ressources de l'unité sont **visibles par l'unité seulement**, sauf partage explicite.
- Les ressources sont **indexées par Holy** dans le périmètre de l'unité, sauf celles marquées confidentielles (§10.2).

**Type spécialisé « Répertoire de chants »** (activable pour les commissions qui en ont besoin, par exemple la louange)
- **Fiche chant** : titre, auteur, **tonalité**, tempo, paroles, partition (PDF), lien audio ou vidéo.
- **Liste de chants** (setlist) rattachée à une répétition ou à un culte. Elle apparaît dans le **programme du culte** et dans le **conducteur** (§7.3, §8.1).
- **Historique automatique** : « chanté 12 fois, dernière fois le 05/10 », pour varier le répertoire.
- **Archive des répétitions** : date, chants travaillés, présents, bilan.
- D'autres types spécialisés pourront être ajoutés plus tard sur le même principe, sans toucher aux autres commissions.

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
  - bouton « Analyser cette zone avec Holy » (synthèse chiffrée locale).
- **Couches activables** :
  - choroplèthe (membres, croissance, taux de présence),
  - clusters de membres,
  - carte de chaleur des nouveaux venus,
  - lieux d'activités,
  - limites des zones.
- **Curseur temporel** : rejouer la croissance de l'église mois par mois (animation).
- **Fond de carte** : OpenFreeMap (tuiles vectorielles, sans clé), décision du 2026-10-03.
- **Mode dégradé** : sans réseau, la carte s'affiche sans fond (limites seules). L'adresse d'un membre peut **proposer** sa zone, jamais l'imposer.
- **Couche cellules** : points ou secteurs des cellules de prière par zone.
- **Confidentialité** : aucune adresse individuelle n'est visible, sauf pour les rôles autorisés. Par défaut, les données sont agrégées à la commune ou au quartier. Géocodage au quartier, pas à la porte.

---

## 3. Programme d'activités et alertes

### 3.1 Modèle
- **Programme** : annuel, trimestriel ou mensuel. Il est rattaché à **n'importe quelle unité de l'organisation** : église, commission, tribu, zone, école… (via `org_unit`). Il contient des **activités**.
  - Les programmes des unités se **consolident** dans le programme de l'église : le pasteur voit tout, chaque berger voit et gère le programme de son unité.
  - Le responsable de l'unité **et ses adjoints** gèrent son programme, ainsi que les membres à qui ils ont **délégué** ce droit (§1.5). Exemple : le groupe de louange planifie ses répétitions et archive ses listes de chants dans son espace.
  - Le responsable de l'unité est propriétaire de son programme. Les activités d'une unité peuvent être soumises à validation de l'échelon supérieur (paramétrable).
- **Activité** :
  - type (paramétrable), titre, description,
  - date et heure de début et de fin, récurrence (RRULE),
  - lieu, commission organisatrice, responsable, intervenants,
  - budget prévisionnel **indicatif** (décision du 2026-10-03 : simple montant informatif, aucun suivi de dépenses ni comptabilité, conformément à « aucune finance »),
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

- Canaux : bandeau INFOS, centre de notifications, push (autorisé le 2026-10-03 : service du téléphone, texte court sans donnée pastorale), e-mail (SMTP configurable), WhatsApp (lien de partage).
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
4. **CR brouillon construit automatiquement** à partir du modèle de CR et des notes structurées point par point (traitement 100 % local, sans génération de texte), au format :
   - en-tête (type, date, lieu, président de séance, secrétaire, présents / absents / excusés),
   - résumé exécutif en 3 à 5 lignes,
   - points traités : discussion → **décision**,
   - **PDA proposés**, saisis dans les notes avec une syntaxe simple (ex. « ACTION : … @porteur avant le 15/11 ») : action, porteur, échéance. Ils sont à valider un par un.
   - points reportés, prochaine réunion.
5. **Relecture et modification** par l'assistante. Le CR est **versionné**.
6. **Validation par la personne qui rédige le CR** (décision du 2026-10-03 : pas de validation par un tiers). Une fois validé, le CR est figé (une nouvelle version reste possible, tracée).
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
- **Identification** : date, site, numéro du culte (1er / 2e… : plusieurs cultes possibles le dimanche, un seul site actif au départ), thème, texte biblique, prédicateur.
- **Horaires** : début et fin prévus / réels (préremplis par le conducteur), durée, retard au démarrage.
- **Effectifs** :
  - par **tranche d'âge paramétrable** (par défaut : enfant < 12 ans, adolescent 12-17, jeune 18-35, adulte 36 et plus), hommes / femmes,
  - total calculé automatiquement,
  - **nouveaux venus / visiteurs**, nouveaux convertis, en ligne (si diffusion live).
  - **Méthode de comptage** : comptage manuel, check-in ou estimation (indice de fiabilité).
- **Service** : nombre de serviteurs mobilisés par commission, absences de serviteurs.
- **Qualité** : note de 1 à 5 par séquence importante, points forts, points à améliorer.
- **Incidents** : technique, sécurité, santé, logistique (catégorie + description), avec possibilité de **créer un PDA** directement.
- **Contexte** : événement exceptionnel (fête, pluie forte, jour férié). C'est utile pour expliquer les variations dans les analyses.
- **Commentaire libre**, puis **résumé chiffré automatique** (effectifs, écarts au prévu, comparaison aux dimanches précédents).
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
- **CR clair construit à partir du modèle et des notes**, modifiable puis validé par son rédacteur,
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
| **Cellules** | Nombre de cellules par zone, participation hebdomadaire, % de membres en cellule |
| **Fidélisation (ADN)** | % de nouveaux venus contactés sous 48 h, taux de retour 1re → 2e visite, délai moyen d'intégration, % intégrés en 90 jours, qui invite le plus |
| **Maturité spirituelle (Suivi des âmes)** | Âmes par mois, % contactées sous 48 h, % baptisées à 6 mois, % consolidées, rétention à 12 mois, charge par accompagnateur |
| **Calling** | Taux de joignabilité, délai de premier contact, appels par appelant, personnes qui ne souhaitent plus être contactées |

Compléments (repères publiés par des outils de pilotage d'église, affichés en légende et **paramétrables**) :
- **Contact sous 48 h** : % de nouveaux venus contactés dans les 48 heures (complète le délai de premier contact, déjà calculé).
- **Taux de retour 1re → 2e visite** : environ 21 % dans les églises en croissance, contre 9 % dans les autres.
- **Visiteurs** : 4 à 5 % de la fréquentation annuelle moyenne en nouveaux visiteurs.
- **Participation aux tribus et cellules** (petits groupes) : cible indicative de 75 % des membres.
- **Ratio de serviteurs** : environ 1 serviteur pour 3 participants.
- **Attrition annuelle** (départs, décès, inactifs) : 5 à 8 % est la norme ; à suivre explicitement.
- **Prochaine étape franchie** : % de visiteurs qui rejoignent une tribu, une cellule, une école ou une commission dans les 90 jours.

Sources : [Churchteams](https://go.churchteams.com/5-health-metrics-pastors-overlook-and-how-to-track-them/), [Vision Room](https://www.visionroom.com/9-numbers-indicate-healthy-church-growth/), [Pushpay](https://pushpay.com/blog/church-metrics-track-track/), [Ministry Brands](https://www.ministrybrands.com/blog/the-most-important-church-metrics-to-track).

### 9.2 Visualisations phares
- **Entonnoir du parcours du membre** : Visiteur → Revenu → Intégré en tribu → Converti / baptisé → Formé (école) → Serviteur → Leader. Pour chaque étape : volumes, taux de passage et durée moyenne. C'est **le** graphique d'une église axée sur la croissance.
- Courbe de fréquentation sur 52 semaines (H/F/E empilés, moyenne mobile, annotations des événements).
- **Carte de chaleur des présences en tribu** (tribus × semaines).
- Carte des zones (choroplèthe de la croissance).
- Liste « **Membres à suivre cette semaine** » (absences, événements de vie, nouveaux venus sans contact), avec une action en un clic.
- Bloc « **À venir sur 14 jours** » (activités, échéances) et « **Alertes** ».
- **Synthèse de la semaine** par Holy, en 5 points chiffrés, en haut du tableau de bord.

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

### 10.2 Architecture (décision de Joel, confirmée le 2026-10-03 : **100 % local**, aucune donnée envoyée à l'extérieur, ARTCI)
- **Chiffres** : l'assistant appelle des **outils** qui exécutent des requêtes paramétrées. Le filtre de périmètre est **injecté côté serveur** à partir de l'identité de l'utilisateur, **jamais à partir du texte de la question**. Aucun chiffre n'est inventé.
- **Textes** : recherche locale (BM25, existante) sur les CR, bilans, commentaires, documents, guides et glossaire. Chaque texte indexé porte des métadonnées de périmètre (`org_unit_id`, niveau de confidentialité). La recherche est filtrée **avant** la récupération.
- **Exclusions** : les données marquées confidentielles et le suivi pastoral nominatif ne sont jamais indexés.
- **Pas de modèle de langage externe** : pas d'API Claude, pas de pgvector. Une couche d'abstraction permet d'ajouter plus tard un modèle **local** sans changer le reste.
- Réponses avec **sources citées et cliquables**. Journal d'audit des questions posées (qui, quand, périmètre).

- **Accueil & suivi (§14)** : Holy ne lit que des **statistiques agrégées** (nombre, taux, délais), jamais les noms ni le journal de suivi des âmes, jamais les résultats d'appels individuels.
- **Toutes les données de l'application** sont accessibles à Holy par des outils de lecture dédiés (membres, présences, nouveaux venus, demandes, rapports et bilans, objectifs, activités, PDA, réunions, documentation et glossaire), **toujours filtrés par le périmètre de l'utilisateur côté serveur** ; aucune donnée hors périmètre ni confidentielle (suivi pastoral nominatif) n'est jamais lue.

### 10.3 Fonctions
- **Questions à Holy** : questions libres et questions suggérées selon le rôle, traitées par les outils chiffrés et la recherche locale. Exemples :
  - Patriarche : « Qui sont les membres de ma tribu absents depuis 3 semaines ? »
  - Responsable de zone : « Compare la présence des tribus de ma zone ce trimestre. »
  - Responsable de commission : « Quels serviteurs de ma commission ont manqué plus de 2 répétitions ce mois-ci ? »
  - Pasteur : « Quelles zones ont la plus forte croissance ? »
  - Pasteur : « Quelles décisions ont été prises aux 3 dernières réunions des bergers ? Quels PDA sont en retard ? »
- **Synthèses automatiques chiffrées** (fondées sur des règles) :
  - synthèse hebdomadaire par rôle (lundi matin),
  - synthèse du culte,
  - synthèse mensuelle de l'église (PDF).
- **Modèles et assistants de saisie** (sans génération de texte) :
  - CR construit à partir du modèle et des notes structurées,
  - PDA saisis dans les notes puis validés,
  - résumé chiffré de bilan,
  - ordre du jour suggéré à partir des données de la semaine.
- **Détection proactive** :
  - baisse de fréquentation d'une zone ou d'une tribu,
  - hausse des absences,
  - commission en sous-effectif,
  - porteur de PDA surchargé,
  - tribu à scinder (ratio trop élevé).
  - Ces détections sont poussées en alertes.
- **Graphiques générés dans la réponse** et export PDF.
- **Évaluation** : jeu de questions de test par rôle, **incluant des tests de fuite de périmètre** (un patriarche demande des données d'une autre tribu → refus attendu) et un test vérifiant qu'aucun appel externe n'est fait.

---

## 11. Menu optimisé (sans répétition)

Les sections suivent le **rythme de vie de l'église**. Le menu reste **entièrement paramétrable** (`menu_config` : visibilité par rôle, ordre, libellés, icônes, feature flags). Le menu ci-dessous est la configuration **par défaut**.

**Règle : un objet = une seule entrée.** Les différentes vues d'un même objet sont des **onglets** ou des **filtres**, jamais des entrées de menu en double. Tout ce qui est « à moi » est regroupé dans **Ma semaine**.

| Section | Entrées |
|---|---|
| **🏠 MON ESPACE** | Tableau de bord · Ma semaine (mes activités, actions, bilans à remplir, **mes appels**, **mes âmes**) · Notifications & alertes |
| **⛪ DIMANCHE** | Programme & conducteur · Bilans du dimanche (culte, tribus) · Réceptions pastorales |
| **🤝 NOUVEAUX & ÂMES** | Nouveaux venus (ADN) · Suivi des âmes · Calling · Suivi pastoral (droits dédiés) |
| **👥 COMMUNAUTÉ** | Membres (fiche, carte, parcours de croissance) · Familles · Carte des zones |
| **🏛️ UNITÉS** | Mes unités · Toutes les unités (filtre : zones, tribus, cellules, commissions, écoles). Chaque unité a son espace à onglets (§1.5) : Aperçu, Bureau, Membres, Activités, Ressources, Réunions & PDA ; les écoles y ajoutent Classes, Présences et Attestations. |
| **📅 PILOTAGE** | Calendrier des activités · Réunions & comptes rendus · Plans d'action · Plannings (cleaning, serviteurs) · Rapports & synthèses |
| **✨ HOLY** | Une seule entrée, plus le bouton flottant présent partout |
| **📣 COMMUNICATION** | Diffusions (listes, e-mail, WhatsApp) · Annonces (bandeau INFOS) |
| **⚙️ ADMINISTRATION** | Organisation (types d'unités, zones, fonctions de bureau) · Rôles, périmètres & délégations · Modèles & types (bilans, CR, réunions, activités, parcours, scripts d'appel, cartes) · Menu & apparence · Imports & sauvegardes |
| **❓ AIDE** | Documentation & glossaire |

**Ce qui a été regroupé par rapport à la version précédente (43 → 29 entrées, avec les nouveaux modules en plus)**
- « Tribus », « Cellules », « Commissions », « Écoles », « Classes », « Présences », « Validation & attestations » → une seule entrée **Unités** à filtre.
- « Bilans de tribu » (Dimanche) et « Tribus » (Communauté) → le bilan de tribu est dans **Bilans du dimanche** ; la tribu est dans **Unités**.
- « Activités des commissions » et « Programme d'activités » → **Calendrier des activités** (filtre par unité) et onglet Activités de chaque unité.
- « Réunions » et « Comptes rendus » → **Réunions & comptes rendus**.
- « Alertes » et « Notifications » → **Notifications & alertes**.
- « Synthèses », « Analyses proactives » et « Chat IA » → **Holy** ; les synthèses chiffrées sont aussi dans **Rapports & synthèses**.
- « Parcours de croissance » est un onglet de la fiche **Membres**.

Principes :
- **3 clics maximum** pour toute action fréquente du dimanche.
- **Chaque utilisateur ne voit que les entrées utiles à son rôle** (par exemple, « Calling » seulement pour les appelants).
- **Barre d'onglets mobile** par rôle. Exemple pour un patriarche : Accueil · Ma tribu · Bilan · Holy · Moi.
- Bouton d'action rapide (+) contextuel : nouveau bilan, nouvelle action, nouveau venu.
- Badges dynamiques sur les entrées : bilans en attente, PDA en retard, appels du jour, alertes.

---

## 12. Exigences transverses
- **Documentation vivante (décision de Joel, 2026-10-03)** : le **glossaire** (`docs/glossaire.md`), l'**aide rapide** (`docs/aide-rapide.md`, une fiche « Comment… ? » par action courante) et les guides (`docs/guide-utilisateur.md`, `docs/guide-admin.md`) sont **mis à jour à chaque modification fonctionnelle, dans le même commit** : nouveau terme, nouvel écran, geste qui change, fonction annoncée devenue disponible (retirer la mention « (bientôt) »). Ils sont consultables dans l'application (menu Aide > Documentation, Glossaire) et **indexés par Holy** ; un test vérifie que la copie lue par Holy est identique aux documents.
- **Holy accède à toutes les données de l'application**, toujours limitées au périmètre de la personne qui pose la question (voir §10) : documentation et glossaire pour l'usage, et outils de lecture pour les membres, présences, nouveaux venus, demandes, rapports, objectifs, activités, PDA, réunions et bilans.
- **Stockage de fichiers** : documents sur le serveur, **10 Mo maximum par fichier**, types autorisés et quota par unité paramétrables (§1.5) ; audio et vidéo par **liens**. Sauvegardes chiffrées incluant les fichiers ; contrôle du type réel du fichier à l'envoi.
- **Hors ligne d'abord pour le dimanche** : bilans, présences, notes de réunion. File de synchronisation et résolution des conflits.
- **Audit** : qui a créé, modifié ou validé quoi, et quand.
- **RGPD / confidentialité** : consentement des membres, droit d'accès et de rectification, minimisation (réceptions pastorales anonymes).
- **Exports** : PDF à la charte de l'église, Excel.
- **i18n** : français par défaut.
- **Performance web** : voir les objectifs Lighthouse dans `CLAUDE.md`. Carte, graphiques lourds et IA chargés en différé.

---

## 13. Questions ouvertes et réponses (2026-10-03)
1. **Berger** : tranché (§0, §1.2). Une tribu **peut** s'étendre sur plusieurs zones.
2. **Volumes** : moins de 2 000 membres, 50 tribus, 30 commissions, 10 écoles.
3. **Cultes** : plusieurs cultes possibles le dimanche ; un seul site actif au départ (multi-sites prévu).
4. **Âges** : enfant < 12, adolescent 12-17, jeune 18-35, adulte 36 et plus, **paramétrable**.
5. **Offrandes** : **aucune finance**, offrandes comprises, nulle part dans l'application.
6. **WhatsApp** : simple **lien de partage** (pas de WhatsApp Business API pour l'instant).
7. **E-mail** : **SMTP configurable** (passerelle existante, ADR 0013).
8. **Validation d'un CR** : par **la personne qui le rédige**, sans tiers.
9. **Commune** : une seule zone à une date donnée, avec historique des redécoupages.
10. **Fond de carte** : OpenFreeMap.
11. **Cellules** : unités de prière hebdomadaire par secteur ou quartier, plusieurs par zone, sous la responsabilité du responsable de zone (§0).
12. **Carte de membre** : matricule `SARO-<année d'arrivée>-<numéro>`, **sans expiration**, QR code personnel réutilisé pour les badges d'activité.
13. **IA** : 100 % locale (§10).

14. **Adjoints** (2026-10-04) : mêmes droits que le responsable par défaut, paramétrable (§1.4).
15. **Ressources** (2026-10-04) : documents sur le serveur, 10 Mo par fichier ; audio et vidéo par liens (§1.5).
16. **Deux équipes distinctes** (2026-10-04) : **ADN** = fidélisation et intégration des nouveaux venus ; **Suivi des âmes** = maturité spirituelle des nouveaux convertis ; une personne peut être suivie par les deux en parallèle (§14.1).
17. **Appelants** (2026-10-04) : chaque source d'appel a ses appelants — ADN pour les nouveaux venus, le patriarche pour ses absents, la commission Suivi des âmes pour les âmes (§14.4).
18. **Ordre des lots** (2026-10-04) : espace d'unité (lot 6 bis) puis Accueil & suivi (lot 6 ter), juste après le lot 6.

À confirmer (non bloquant) : le responsable de zone voit-il en lecture seule les nouveaux venus de sa zone (§14.2), et le nombre d'âmes de sa zone (§14.3) ? Quel quota d'espace par unité par défaut (§1.5) ?

Points encore ouverts (non bloquants) : test sur un téléphone Android (build de développement ou EAS) ; image de la signature du pasteur (à charger dans le modèle de carte) ; compte Apple pour iOS (plus tard).

---

## 14. Nouveaux & âmes : accueil, suivi et calling

> Cette section regroupe **deux équipes et un outil commun**. Le principe : **mêmes personnes, mêmes appels, un seul moteur**, mais **deux responsabilités distinctes**.

### 14.1 Principe : deux équipes, deux parcours, un moteur d'appels

| | **ADN** (Accueil des Nouveaux) | **Suivi des âmes** |
|---|---|---|
| **Personnes suivies** | Les **nouveaux venus** | Les **nouveaux convertis** (âmes) |
| **Objectif** | **Fidélisation et intégration** à l'église | **Maturité spirituelle** |
| **Fin du parcours** | *Intégré* (devient membre) | *Consolidé* |
| **Qui fait le travail** | Les membres de la commission ADN et leur responsable (berger) | Les membres de la commission Suivi des âmes et leur responsable (berger) |

- **Les deux suivis sont parallèles, pas successifs.** Un nouveau venu qui prend la décision de suivre Christ **reste suivi par ADN** pour son intégration **et entre aussi** dans le suivi des âmes pour sa maturité. Un membre ancien qui se convertit entre dans le suivi des âmes **seul**.
- **Pas de double appel** : sur la fiche d'une personne, chaque équipe voit uniquement « **dernier contact : il y a 2 jours, par ADN** » (date et équipe, sans contenu), pour se coordonner.
- **Le contenu d'un suivi reste dans son équipe** : ADN ne lit pas le journal des âmes, et réciproquement.
- Les noms de personnes ne sont pas écrits dans cette spécification : les responsables sont désignés **par leur fonction**.

### 14.2 ADN : suivi des nouveaux venus (fidélisation et intégration)
Les repères de référence (contact sous 48 h, plusieurs prises de contact la première semaine, une personne attitrée, une prochaine étape claire) sont ceux des indicateurs du §9.1.

**Saisie**
- **En 30 secondes**, par l'accueil le dimanche, **hors ligne**. Une **fiche d'accueil par QR code** peut aussi être remplie par le visiteur lui-même.
- Champs : nom, prénom, téléphone / WhatsApp, commune et quartier (→ zone proposée), comment il a connu l'église, **invité par** (un membre, choisi dans le registre), besoin de prière, **accord pour être recontacté**.
- **Détection automatique des doublons** par numéro de téléphone.

**Parcours** (étapes paramétrables)
1re visite → Contacté → 2e visite → 3e visite → Orienté (tribu, cellule, école) → **Intégré** (statut Membre, matricule et carte de membre). Sortie : *Sans suite*, avec un motif.

**Automatismes**
- Le **responsable ADN répartit** les nouveaux venus entre les membres de la commission (affectation manuelle, ou proposée selon la charge).
- Orientation **proposée** d'après la commune : zone, tribu, cellule (jamais imposée).
- Le membre qui a **invité** la personne est proposé comme **parrain**.
- Appels générés automatiquement à **J+1, J+7 et J+30** (§14.4).
- Règle déjà prévue : **présent 2 fois** → proposé à l'intégration (§8.3).
- Si la personne prend une décision pour Christ → **proposition d'entrée dans le suivi des âmes** (le suivi ADN continue).

**Droits**
- Le **responsable ADN** voit tous les nouveaux venus et les statistiques.
- Un **membre ADN** ne voit que les personnes qui lui sont assignées (nom, téléphone, script d'appel, historique de contact).
- Le **responsable de zone** voit, en **lecture seule**, les nouveaux venus de sa zone **[À CONFIRMER]**.

**Indicateurs** : voir le §9.1 (famille Fidélisation).

### 14.3 Suivi des âmes : maturité spirituelle des nouveaux convertis

**Entrée d'une âme** : décision au culte (le bilan du culte compte déjà les nouveaux convertis ; ici on les identifie), lors d'une évangélisation, ou depuis le suivi ADN.

**Un accompagnateur par âme**, choisi parmi les membres de la commission Suivi des âmes. **Charge maximale paramétrable** (par exemple 5 âmes par accompagnateur) ; alerte si elle est dépassée.

**Parcours de maturité** (étapes paramétrables, par défaut) :
1. Premier contact sous 48 h.
2. Rencontre ou visite.
3. **Enseignements de fondement**, par une **école** du §8.5 (« École des nouveaux convertis »), pour réutiliser présences et attestation.
4. Baptême.
5. Intégration en tribu ou en cellule.
6. Inscription à l'école de disciples.
7. Première responsabilité de service (commission).

**Repères de maturité** (simples, calculés à partir de données existantes) : assiduité au culte, en tribu ou en cellule, assiduité à l'école, étapes franchies. Aucun jugement spirituel n'est enregistré par l'application.

**Journal de suivi** : date, type (appel, visite, rencontre, prière) et **résultat court**. **Les confidences n'y vont jamais** : elles relèvent du **suivi pastoral chiffré** existant (§1.1 bis). La frontière entre les deux est affichée dans l'écran de saisie.

**Statuts** : Actif · En pause · Perdu de vue (après N tentatives sans réponse) · **Consolidé** (parcours terminé).

**Alertes** : âme sans contact depuis 7 jours, étape bloquée depuis plus de 30 jours, accompagnateur surchargé.

**Droits**
- Le **responsable du suivi des âmes** voit toutes les âmes et les statistiques, et assigne les accompagnateurs.
- Un **accompagnateur** ne voit que **ses** âmes.
- Le **responsable de zone** voit uniquement le **nombre** d'âmes de sa zone, pas les noms. **[À CONFIRMER]**

**Indicateurs** : voir le §9.1 (famille Maturité spirituelle).

### 14.4 Calling : moteur d'appels commun

**Un seul moteur, plusieurs sources, chacune avec ses appelants :**

| Source des appels | Qui appelle |
|---|---|
| Nouveaux venus (J+1, J+7, J+30) | Les membres de la commission **ADN** |
| Âmes à suivre | Les accompagnateurs de la commission **Suivi des âmes** |
| Absents depuis 3 dimanches (§8.3) | Le **patriarche** de la tribu |
| Campagnes ponctuelles (rappel d'une activité, invitation…) | L'équipe désignée par celui qui lance la campagne |

- **Listes d'appels générées automatiquement**, répartition par le responsable de la source. Un responsable peut **déléguer ponctuellement** une liste à un membre (§1.5).
- **Script d'appel paramétrable**, affiché pendant l'appel, par source.
- Boutons **Appeler** et **WhatsApp** (lien de partage) en un geste.
- **Résultat en un clic** : Joint · Pas de réponse · Mauvais numéro · Rappeler le… · Souhaite une visite · Demande de prière · **Ne souhaite plus être contacté**. La prochaine action est créée automatiquement.
- Après **3 tentatives sans réponse**, la personne remonte au responsable de la source.
- **« Ne souhaite plus être contacté » est toujours respecté** : la personne disparaît de toutes les listes d'appel.
- Un **appelant** ne voit **que les personnes de sa liste** (nom, téléphone, script), pas le reste de la fiche.
- **Hors ligne** : les listes du jour sont disponibles sans réseau, les résultats se synchronisent ensuite.

### 14.5 Confidentialité de ces modules
- **Minimisation** : seules les informations nécessaires au suivi sont enregistrées. Consentement de recontact tracé.
- **Notifications** : jamais de nom ni de donnée pastorale dans le texte d'une notification push.
- **Journal d'accès** : chaque consultation d'une fiche de nouveau venu ou d'âme est journalisée.
- **Holy** : statistiques agrégées uniquement (§10.2).
- **Droit à l'effacement et à l'export** pour toute personne suivie (§12, ARTCI).
