# Intégration de la SPEC v3 (2026-10-04) — à donner à Claude Code

## Étape 1 — Remplacer les fichiers
- `docs/specs/SPEC_FONCTIONNELLE.md` : remplacer par la nouvelle version (v3).
- `CLAUDE.md` : remplacer par la nouvelle version.
- Faire un commit « Spec v3 : espace d'unité, bureau, ressources, ADN, suivi des âmes, calling, menu sans répétition » avant de lancer le prompt ci-dessous.

## Étape 2 — Prompt (mode plan : Shift+Tab, puis coller)

```
J'ai mis à jour la SPEC (v3) et CLAUDE.md. Ne code rien : analyse et plan.

Lis dans la SPEC : §0 (glossaire), §1.1 bis, §1.2, §1.4, §1.5, §3.1, §9.1 (nouvelles lignes), §10.2, §11 (menu), §12, §13 (réponses 14 à 18) et §14 en entier.

1. Mets à jour docs/specs/PLAN_LOTS.md : ajoute le lot 6 bis (espace d'unité : registre des personnes à statut, bureau à mandats, délégations, ressources, répertoire de chants) et le lot 6 ter (accueil & suivi : ADN, suivi des âmes, calling, règle « nouveau venu présent 2 fois »). Pour chacun : modèle de données, endpoints, écrans, tests (dont fuites de périmètre), critères d'acceptation, découpage en sous-lots.
2. Compare avec le code existant et liste les CONFLITS et les risques, en particulier :
   - la table des nouveaux venus actuelle (à réutiliser, pas à dupliquer) et le registre des membres : migration réversible vers un registre unique à statut ;
   - la règle « toute personne désignée est un membre » : repère tous les champs existants qui stockent un nom en texte libre (porteurs, intervenants, responsables…) et propose leur migration vers une référence à un membre ;
   - le menu : migration de `menu_config` vers les 29 entrées de la SPEC §11 (correspondance ancien → nouveau, sans perdre les personnalisations existantes) ;
   - le stockage de fichiers : 10 Mo par fichier, quota par unité, limite de taille côté Caddy, types contrôlés, sauvegardes incluant les fichiers (propose un ADR) ;
   - les droits : responsable ET adjoint (mêmes droits par défaut, paramétrable), délégations datées et révocables, périmètre « appelant » limité à sa liste ;
   - Holy : statistiques agrégées uniquement pour les modules du §14, jamais le journal de suivi des âmes ;
   - le mode hors ligne des listes d'appels.
3. Liste la documentation vivante à mettre à jour quand chaque sous-lot sera livré (glossaire : unité, bureau, nouveau venu, ADN, âme, suivi des âmes, calling ; aide-rapide ; guides), avec la mention « (bientôt) » d'ici là.
4. Pose-moi les questions bloquantes (SPEC §13, « À confirmer ») et attends ma validation. Aucun nom de personne réelle dans le dépôt : désigne les responsables par leur fonction.
```

## Étape 3 — Après validation
Un lot par conversation : `/clear`, puis « Lance le lot 6 bis, sous-lot a » (le lot 6 doit être terminé avant).
