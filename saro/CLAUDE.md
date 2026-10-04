# CLAUDE.md — SARO Manager (Sacerdoce ROyal)

> Mémoire du projet, chargée à chaque session : elle reste COURTE et à jour.
> Le détail métier est dans @docs/specs/SPEC_FONCTIONNELLE.md : le lire avant toute tâche fonctionnelle.
> En cas de conflit avec un prompt de tâche, ce fichier et la SPEC priment, sauf instruction explicite de Joel.

## 1. Vision
Application de gestion d'une église **axée sur la croissance et le suivi des membres**, en Côte d'Ivoire avec une diaspora (France…). Web installable et Android (iOS plus tard). Une église, plusieurs sites, environ 200 utilisateurs actifs, hébergement sur VPS.
Domaines : organisation (zones, tribus, cellules, commissions, écoles) avec un **espace par unité**, dimanche, semaine, pilotage (activités, réunions, PDA, alertes), **accueil et suivi** (ADN, suivi des âmes, calling), assistant « Holy » limité au périmètre de chacun.
- **Règle d'or : tout est paramétrable** (types, templates, zones, rôles, seuils, menus). Rien de métier codé en dur ; back office par interface, jamais par le code ou la base.
- **L'expérience utilisateur prime** : peu de clics (3 maximum pour le dimanche), libellés clairs en français, erreurs utiles et bienveillantes, états vides explicatifs, squelettes, confirmation avant suppression avec annulation, validation en ligne, clavier et mobile, accessibilité AA.

## 2. Dépôt et stack (réelle)
- `backend/` : Python 3.13, FastAPI, SQLAlchemy 2, Alembic, Pydantic v2 ; couches API / services ; planificateur intégré (pas de Redis). Base SQLite (WAL, clés étrangères) par défaut, PostgreSQL 16 testé (ADR 0002). Lancement : voir `README.md`.
- `client/` : nouvelle interface Expo SDK 57 + Expo Router, TypeScript strict, NativeWind 4, Reanimated 4 sur Android (version allégée en CSS sur le web, ADR 0017), TanStack Query, Zustand, formulaires en état local avec validation en ligne, FlashList (ADR 0015).
  - Graphiques (ADR 0016) : SVG du navigateur sur le web (**aucun Skia/CanvasKit sur le web**), Victory Native XL/Skia sur Android.
  - Variantes de plateforme : `fichier.ts` (natif) / `fichier.web.ts` (web), ex. `src/lib/download*.ts`, `src/features/dashboard/useLive*.ts`.
  - Architecture : `src/features/<module>/`, composants partagés `src/components/ui/`, icônes à l'unité via `src/components/ui/lucide.ts`.
  - Couleurs lues en base (écran Apparence) ; `src/theme/tokens.ts` = valeurs de secours.
  - Tests : `npm test` (Jest), `npm run typecheck`, `npm run lint`, Playwright `client/e2e/` sur l'export web (`npm run build:web`, `node scripts/serve-web.mjs 8082`).
- Ancienne interface Angular (`frontend/`) **retirée** le 2026-10-03 (décision de Joel, ADR 0021) : tout est en React (Expo).
- Déploiement : Docker Compose (API, Caddy HTTPS + CSP stricte, sauvegarde chiffrée quotidienne, restauration testée), CI GitHub.
- Polices et ressources auto-hébergées, aucun CDN.

## 3. Charte graphique
- Couleurs **SARO par défaut** : bleu profond `#073B4C` (institutionnel), or `#D4AF37` (actif, filets, avec parcimonie), ivoire `#FAF8F2` (fond), bordeaux `#8B0000` (alertes), anthracite `#1F2933` (texte). Titres courts en Cinzel, interface en Inter.
- Écran Apparence : préréglages « SARO royal » (non supprimable), « Orange vif » (palette des applications de référence : `#FF7900`, barre `#1A1A1A`, fond `#F4F4F4`), thèmes personnalisés ; contraste AA vérifié ; clair et sombre.
- Mise en page inspirée de BILLING / ARIA / ANITA (`ressource-joe/`) : barre latérale sombre par sections avec élément actif en bloc d'accent, bandeau INFOS défilant, cartes KPI à compteur animé, onglets soulignés, bouton Holy flottant, connexion sombre avec diagramme circulaire des modules.
- Logo source : `assets/logo-saro-source.jpg` (symbole : l'Arche de l'Alliance). **Jamais le logo ni le nom d'une autre organisation.**

## 4. Décisions prises (Joel)
- Refonte Expo, un seul code web + Android ; web et Android d'abord (2026-10-02).
- **IA 100 % locale** (2026-10-02, reconfirmé le 2026-10-03) : aucune donnée envoyée à l'extérieur (ARTCI). Chiffres via des outils filtrés par périmètre côté serveur, textes via recherche locale (BM25), CR construits à partir de modèles et de notes structurées, pas de génération de texte. Couche d'abstraction gardée pour un modèle local futur.
- **Aucune finance** (dîmes, offrandes, dons, budget de l'église), confirmé le 2026-10-03.
- **Organisation générique historisée** `org_unit` (2026-10-03), migration réversible au lot 4.
- **Bergers** = tous les responsables de premier niveau (zone, tribu/patriarche, commission, assistant du pasteur) ; périmètre déduit automatiquement des unités dirigées (`org_unit.responsable_id`).
- **Appartenances** : au plus une tribu et une zone par membre à une date donnée ; une **tribu peut chevaucher plusieurs zones** (le responsable de zone voit les membres de sa zone) ; une commune dans une seule zone à une date donnée ; **cellules** de prière hebdomadaire, plusieurs par zone, sous la responsabilité du responsable de zone.
- **Carte de membre** (SPEC §1.3) : matricule `SARO-<année d'arrivée>-<numéro>`, **sans expiration**, QR code personnel signé (vérification, pointage, badges d'activité).
- Réponses SPEC §13 (2026-10-03) : < 2 000 membres ; plusieurs cultes le dimanche, un site actif ; tranches d'âge paramétrables (enfant < 12, ado 12-17, jeune 18-35, adulte 36+) ; WhatsApp par lien de partage ; e-mail par SMTP configurable ; un CR est validé par son rédacteur ; fond de carte OpenFreeMap.
- Menu entièrement paramétrable (`menu_config` versionné, rôles, sections, modules, onglets mobiles). **Un objet = une seule entrée** : les vues sont des onglets ou des filtres (SPEC §11, 29 entrées).
- **Espace d'unité** (2026-10-04, SPEC §1.4-§1.5) : chaque unité (zone, tribu, cellule, commission, école) a le même espace à onglets (Aperçu, Bureau, Membres, Activités, Ressources, Réunions & PDA). Responsable **et adjoints** (mêmes droits par défaut, paramétrable) gèrent l'espace ; délégations de droits précises, datées, révocables, tracées. **Bureau** à mandats datés historisés.
- **Toute personne désignée est un membre** (2026-10-04) : bureau, porteur de PDA, intervenant, accompagnateur, appelant, délégué… se choisissent dans le registre des membres, **aucune saisie libre de nom**, contrainte en base. **Registre unique des personnes** à statut (nouveau venu → membre → inactif → parti) ; la table des nouveaux venus actuelle est réutilisée, pas dupliquée.
- **Ressources d'unité** (2026-10-04) : documents sur le serveur, **10 Mo par fichier**, quota par unité ; audio et vidéo **par liens** ; type spécialisé « Répertoire de chants » (tonalité, setlist liée au culte, historique) pour la louange.
- **Accueil & suivi** (2026-10-04, SPEC §14) : **deux équipes distinctes** — commission **ADN** (nouveaux venus : fidélisation, intégration) et commission **Suivi des âmes** (nouveaux convertis : maturité spirituelle) ; suivis **parallèles**, contenu non partagé, seul « dernier contact » est visible des deux ; **moteur d'appels commun** (calling) où chaque source a ses appelants (ADN, patriarche pour ses absents, accompagnateurs). Le journal de suivi des âmes ne contient jamais de confidences (elles vont au suivi pastoral chiffré) ; Holy ne lit que des statistiques agrégées sur ces modules.
- PDA : table unique `action`, origine polymorphe, jamais clôturée automatiquement.
- Bilans : templates versionnés, réponses JSON + indicateurs clés en table de faits.
- Zones : composition en communes historisée (`valid_from` / `valid_to`), géométrie calculée par union.
- Réceptions pastorales : statistiques uniquement, aucun nom ni contenu. Le suivi pastoral nominatif existant reste chiffré et à droits dédiés.
- Dimanche hors ligne d'abord (présences, bilans, notes de réunion).
- Tranchés aussi : traduction reportée (ADR 0009), fonds de carte OpenStreetMap + mode sans fond, limites de communes en données ouvertes locales, sauvegardes sans copie hors site pour l'instant, listes officielles saisies par Joel (écran Organisation).
- Ouverts (non bloquants) : test sur un téléphone Android (build de développement ou EAS), image de la signature du pasteur, compte Apple (iOS, plus tard).
- Mise en production : Claude prépare et teste en local (image, Caddy, guide de bascule et de retour arrière), **Joel déploie** sur le VPS.
- Autonomie accordée le 2026-10-03 : lot 3 en entier puis lot 4, commits par sous-lot, récapitulatif à la fin.
- Lot 6 (2026-10-03) : découpage validé 6a activités + alertes, 6b PDA, 6c réunions + CR + PDF, 6d cleaning ; **budget d'activité indicatif seulement** (un montant prévisionnel informatif, aucun suivi de dépenses ni comptabilité) ; **notifications push autorisées** via le service du téléphone (Google pour Android), avec un texte court sans donnée pastorale ; bandeau INFOS jamais masqué (PAROLE : versets et infos de l'église quand rien n'est à signaler) ; Lighthouse traité en dernier.
- **Angular retiré** (2026-10-03) : l'interface est uniquement React (Expo) ; retirer tout code, document ou plan qui ne sert qu'à l'ancienne interface. Priorité au dynamisme de l'interface.
- Décisions du 2026-10-03 (après le lot 4) : Lighthouse → **micro-optimisations** (option D de `docs/performance.md`, 2 à 3 jours, gain incertain) ; **bascule de production dès maintenant** (l'application n'est pas encore utilisée ; Joel déploie avec `SARO_WEB=client`) ; essai Android **reporté**.

## 5. Performance et qualité
- Lighthouse (export web de production compressé, page de connexion) : objectifs **ordinateur ≥ 90, téléphone ≥ 75**, animations fluides. Mesuré (2026-10-03, après micro-optimisations) : téléphone 54-55, ordinateur 75-82, freinés par le temps de blocage JavaScript (`docs/performance.md`).
- Leviers obligatoires avant la bascule : rendu statique Expo Router (connexion, pages publiques), chargement différé de chaque écran, carte / assistant / temps réel / animations non essentielles en différé, rapport de bundle (`client/scripts/bundle-report.cjs`) à chaque lot.
- API : p95 < 200 ms, pagination serveur partout, ETag.
- Sécurité : OWASP ASVS 2, Argon2id, sessions courtes à rotation, CSP stricte, chiffrement des champs pastoraux, export et effacement des données d'un membre, législation ivoirienne (ARTCI, `docs/conformite.md`).

## 6. Feuille de route
1. Design system, thème, barre latérale, menu paramétrable, connexion — ✅ (2026-10-02)
2. Tableau de bord animé — ✅ (2026-10-03) ; les KPI de croissance (SPEC §9) arrivent avec les lots qui créent leurs données
3. Migration des 31 écrans Angular + passe d'optimisation web + export mobile, puis bascule de production — ✅ (2026-10-03) : parité complète (3a à 3f), export par la feuille de partage d'Android, Reanimated allégé sur le web (ADR 0017), écran d'accueil instantané, image Docker `client/` (Joel déploie, `docs/exploitation.md` §5 bis). Lighthouse connexion : téléphone 64, ordinateur 75, sous les objectifs ; alternatives chiffrées dans `docs/performance.md`, **décision de Joel attendue** (option A recommandée). Android pas encore essayé sur un vrai téléphone.
4. Socle organisationnel, zones historisées, carte, périmètres des bergers, carte de membre (SPEC §1, §2) — ✅ (2026-10-03) :
   - 4a-4b : `org_unit` générique et historisé, cellules, communes avec date d'effet, bergers automatiques, écran en arbre (ADR 0018) ;
   - 4c : carte MapLibre + OpenFreeMap sur le web, chargée à la demande, mode dégradé, panneau de zone, curseur des 12 mois ; SVG sur Android (ADR 0019) ;
   - 4d : carte de membre (matricule jamais réutilisé, QR signé et révocable, impression CR80 et planche A4, carte numérique, scan avec pointage, badges, ADR 0020).
5. Dimanche : programme et conducteur du culte, bilans de culte et de tribu, réceptions, hors ligne (SPEC §7.3, §8.1-8.4) — ✅ (2026-10-03) : 5a programme + conducteur, 5b bilan du culte + `fact_attendance`, 5c appel de tribu + alertes d'absence, 5d réceptions anonymes ; reste la règle « nouveau venu présent 2 fois » (avec le pointage des visiteurs)
6. Pilotage : programme d'activités + moteur d'alertes, PDA unifiés, cleaning, réunions + CR + diffusion PDF (SPEC §3-§5)
6 bis. **Espace d'unité** : registre des personnes à statut, bureau à mandats, délégations, ressources et répertoire de chants (SPEC §1.4-§1.5) — prérequis du lot suivant
6 ter. **Accueil & suivi** : ADN (nouveaux venus), suivi des âmes, calling, règle « nouveau venu présent 2 fois » (SPEC §14) — juste après le lot 6, décision de Joel (2026-10-04)
7. Bilans paramétrables (constructeur) + commissions (SPEC §6, §7)
8. Écoles et parcours de croissance (SPEC §8.5, §9.2)
9. Assistant Holy 100 % local : synthèses, détection proactive, tests de fuite (SPEC §10)
Plan détaillé des lots (modèle, endpoints, écrans, tests, critères) : `docs/specs/PLAN_LOTS.md`.

## 7. Règles de travail
- Avant chaque lot : lire la SPEC concernée, proposer un plan (modèle de données, API, écrans, tests) et **attendre la validation de Joel**. Un lot à la fois.
- Ne jamais inventer une règle métier : poser la question (SPEC §13). Toute personne désignée quelque part se choisit dans les membres (SPEC §1.4) : ne jamais ajouter de champ « nom » libre. Décisions d'architecture : un ADR dans `docs/adr/`.
- Migrations réversibles (montée/descente/remontée testées) ; ne jamais supprimer de données historiques.
- Le filtrage par périmètre est appliqué côté serveur et testé (fuites) pour chaque endpoint, export, indicateur, la carte et l'assistant.
- **Documentation vivante (obligatoire)** : toute modification fonctionnelle met à jour, dans le même commit, `docs/glossaire.md`, `docs/aide-rapide.md` et le guide concerné (retirer « (bientôt) » quand une fonction arrive), puis `python -m app.cli sync-knowledge` pour que Holy les lise (test de synchronisation dans `tests/test_assistant.py`). Holy doit pouvoir lire toutes les données de l'application, toujours dans le périmètre de l'utilisateur (SPEC §10, §12).
- Après chaque lot : lint + typecheck + tests (pytest, Jest, Playwright 390×844 et 1440×900 avec captures), Lighthouse si le web est touché, documentation (guides, `docs/backlog.md`), mise à jour de ce fichier (§4, §6) et de la SPEC si une règle a changé, données de démonstration FICTIVES, récapitulatif pour Joel.
- Aucune vraie donnée de membre dans le dépôt ; secrets uniquement en variables d'environnement (`.env` ignoré, `.env.example` versionné).
- Commits clairs et atomiques. Textes de l'interface en français.

## 8. Historique
- Phases 0 à 4 (2026-10-02, interface Angular + backend) : socle, quotidien, pilotage, transformation et ouverture. Détail : `docs/backlog.md`, ADR 0001 à 0014.
- Lots 3 et 4 (2026-10-03) : parité complète de la nouvelle interface, optimisation web (ADR 0017), bascule préparée (Joel déploie), organisation historisée (ADR 0018), carte (ADR 0019), carte de membre (ADR 0020). Qualité : tests backend sur SQLite et PostgreSQL 16, migrations montée/descente/remontée, parcours Playwright × 2 formats.
- Refonte Expo (ADR 0015, 0016) : étapes 0 à 2 faites ; qualité à fin d'étape 2 : 178 tests backend (SQLite ; PostgreSQL non relancé, Docker indisponible sur le poste), 15 tests Jest, 10 parcours Playwright ; Android non testé sur appareil.
