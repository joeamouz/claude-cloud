# CLAUDE.md — SARO Manager (application de gestion d'église)

> Mémoire du projet pour Claude Code. Ce fichier doit rester COURT et à jour : il est chargé à chaque session.
> Le détail métier est dans @docs/specs/SPEC_FONCTIONNELLE.md : lis-le avant toute tâche fonctionnelle.
> Les sections marquées (à vérifier) sont à confirmer en lisant le code. Corrige-les si elles sont fausses.

## 1. Vision
Application de gestion d'une église **axée sur la croissance et le suivi des membres**, en Côte d'Ivoire avec une diaspora (France, etc.).
Elle couvre :
- l'organisation (zones, tribus, commissions, écoles),
- le dimanche (culte, bilans, réceptions pastorales, écoles, réunion des bergers),
- la semaine (activités des commissions, cleaning, diffusion du programme),
- le pilotage (programme d'activités, réunions, PDA, alertes),
- une **IA de synthèse limitée au périmètre de chaque utilisateur**.

**Règle d'or : tout est paramétrable** (types, templates, zones, rôles, seuils, menu). Rien de métier n'est codé en dur.

## 2. Dépôt et stack
- Racine : `saro-manager/`. Frontend React Native : `client/` (Expo + Expo Router, TypeScript strict).
- Ancien frontend : **Angular**, encore en place pour le web (à vérifier : chemin et statut de la bascule).
- Backend / API / base de données : (à vérifier : langage, framework, BDD, ORM). PostgreSQL + PostGIS + pgvector recommandés pour la carte et le RAG.
- Cibles : **Web** (react-native-web) + **Android**. iOS plus tard.
- Librairies retenues :
  - Expo Router,
  - Reanimated / Gesture Handler / Moti,
  - TanStack Query + Zustand,
  - FlashList, React Hook Form + Zod.
  - Graphiques : Skia sur Android, librairie SVG légère sur le web (fichiers `.web.tsx`).
  - Carte : MapLibre (web : maplibre-gl, Android : @maplibre/maplibre-react-native).
- Convention multiplateforme : `fichier.ts` (natif) et `fichier.web.ts` (web). Exemple : `src/lib/download.ts` et `src/lib/download.web.ts`. L'export de fichiers sur mobile est prévu à l'étape 3.
- Architecture par fonctionnalités : `client/src/features/<module>/`. Composants partagés dans `client/src/components/`. Thème dans `client/src/theme/tokens.ts`.

## 3. Charte graphique (inspirée des applications internes de référence)
- Primaire `#FF7900` (actif, boutons, bandeau INFOS), survol `#F16E00`.
- Barre latérale `#1A1A1A`, titres de section `#8F8F8F` (petites majuscules), texte `#E6E6E6`.
- Fond `#F4F4F4`, cartes `#FFFFFF` (rayon 8 px), texte `#000000` / `#595959`.
- Séries de graphiques : `#FF7900`, `#4BB4E6`, `#50BE87`, `#A885D8`, `#FFD200`.
- Statuts : succès `#32C832`, info `#527EDB`, alerte `#FFCC00`, erreur `#CD3C14`.
- Thème sombre : fond `#0F0F0F`, cartes `#1C1C1C`, halo orange.
- Éléments d'interface signature :
  - barre latérale sombre par sections, élément actif en bloc orange,
  - bandeau INFOS défilant,
  - cartes KPI avec icône carrée et compteur animé,
  - onglets soulignés en orange,
  - bouton assistant IA flottant avec mascotte,
  - page de connexion sombre avec diagramme circulaire animé des modules.
- **Ne jamais utiliser le logo ni le nom Orange.** Utiliser l'identité de l'église.

## 4. Décisions déjà prises
- Refonte du frontend en React Native (Expo), un seul code pour le web et Android.
- **Menu entièrement paramétrable** (`menu_config` côté serveur : visibilité par rôle, ordre, libellés, icônes, sections, feature flags). Menu par défaut : SPEC §11.
- Connexion Microsoft / Google : faite sur le web, à ajouter sur Android plus tard.
- Notifications : centre de notifications + push (Expo) + temps réel (WebSocket/SSE) + e-mail / WhatsApp. Alertes calculées par des jobs serveur.
- IA : API Claude.
  - Les **chiffres** passent par des outils SQL.
  - Les **textes** passent par le RAG (pgvector).
  - Le **périmètre est appliqué côté serveur**, jamais déduit du prompt.
- PDA : **table unique `action`** avec une origine polymorphe (activité, réunion, bilan, cleaning, IA…). Une action n'est jamais clôturée automatiquement.
- Bilans : moteur de templates versionnés. Réponses en JSON + **indicateurs clés en colonnes typées / table de faits** pour l'analyse.
- Zones : composition en communes **historisée** (`valid_from` / `valid_to`) ; la géométrie est calculée par union.
- Réceptions pastorales : **statistiques uniquement**, aucun nom ni contenu.
- Le dimanche fonctionne **hors ligne d'abord** (présences, bilans, notes de réunion).

## 5. Performance (Lighthouse)
- Mesures après l'étape 1 :
  - téléphone **41**, ordinateur **66**, accessibilité 100,
  - ancienne interface Angular : 93,
  - bundle ramené de 5,3 Mo à 2,3 Mo, polices de 1,4 Mo à 60 Ko.
  - Détails : `docs/performance.md`.
- Objectifs : **ordinateur ≥ 90, téléphone ≥ 75** (à confirmer par le propriétaire du projet).
- Leviers obligatoires :
  - rendu statique Expo Router pour les pages publiques et la connexion,
  - chargement différé de chaque écran,
  - **pas de Skia/CanvasKit sur le web**,
  - carte, IA et temps réel chargés en différé,
  - analyse du bundle à chaque étape.

## 6. Feuille de route
1. Design system + thème + barre latérale + menu paramétrable + connexion — ✅ fait (performance web à optimiser)
2. Tableau de bord animé orienté croissance (SPEC §9) — en cours
3. Migration des modules existants + passe d'optimisation web + export mobile
4. Socle organisationnel : `org_unit`, zones et carte, rôles × périmètres (SPEC §1, §2)
5. Dimanche : programme et conducteur du culte, bilans de culte et de tribu, réceptions (SPEC §7.3, §8)
6. Pilotage : programme d'activités + alertes, PDA unifiés, cleaning, réunions + CR IA + diffusion PDF (SPEC §3, §4, §5)
7. Bilans paramétrables (form builder) + commissions (SPEC §6, §7)
8. Écoles et parcours de croissance (SPEC §8.5, §9.2)
9. Assistant IA limité au périmètre + synthèses + détection proactive (SPEC §10)

## 7. Règles de travail pour Claude
- Avant chaque lot : lire la SPEC concernée, proposer un plan (modèle de données, API, écrans, tests) et **attendre la validation**.
- Poser les questions de la SPEC §13 dès qu'elles bloquent. Ne pas inventer de règle métier.
- Toute migration de base de données doit être réversible. Ne jamais supprimer de données historiques.
- Le filtrage par périmètre est testé dans les tests automatisés pour chaque endpoint, chaque export et chaque outil IA.
- Après chaque lot :
  - lint + typecheck + tests,
  - mesure Lighthouse si le web est touché,
  - mise à jour de ce fichier (section 6 et décisions) **et** de la SPEC si une règle a changé.
- Commits clairs, un lot = une branche ou une série de commits cohérente.
- Textes de l'interface en français.
