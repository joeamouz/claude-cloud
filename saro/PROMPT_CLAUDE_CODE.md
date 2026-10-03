# Méthode pour transmettre la spécification à Claude Code

## Étape 1 — Copier les fichiers dans le projet
Dans `C:\JOE_DOC\DEV\SARO\saro-manager\` :
- `docs/specs/SPEC_FONCTIONNELLE.md` → à copier tel quel (créer le dossier `docs/specs` si besoin)
- `CLAUDE.md` → **ne pas écraser le CLAUDE.md existant**. Enregistrer ce fichier sous `docs/specs/CLAUDE_PROPOSE.md` : Claude Code fera la fusion (étape 2).

## Étape 2 — Prompt n°1 : intégration et cadrage (passer en mode plan avec Shift+Tab avant d'envoyer)

```
Nouvelle version des besoins métier. Ne code rien pour l'instant.

1. Lis docs/specs/SPEC_FONCTIONNELLE.md en entier, puis docs/specs/CLAUDE_PROPOSE.md.
2. Fusionne CLAUDE_PROPOSE.md dans notre CLAUDE.md existant :
   - garde tout ce qui est juste dans l'actuel (chemins, commandes, stack réelle, décisions de performance),
   - corrige les sections marquées "(à vérifier)" d'après le vrai code,
   - garde CLAUDE.md court : il doit renvoyer vers la SPEC avec @docs/specs/SPEC_FONCTIONNELLE.md,
   - montre-moi le diff avant d'enregistrer, puis supprime CLAUDE_PROPOSE.md.
3. Analyse la SPEC face au code actuel et donne-moi :
   - les écarts et les risques (modèle de données, performance web, hors ligne, sécurité des périmètres),
   - tes propositions d'amélioration,
   - les questions de la section 13 auxquelles tu as besoin d'une réponse, et toute autre question bloquante.
4. Propose un découpage en lots, dans l'ordre de la feuille de route de CLAUDE.md §6. Pour chaque lot, indique :
   le modèle de données, les endpoints, les écrans, les tests (dont les tests de périmètre) et les critères d'acceptation.
5. Pour le tableau de bord (lot en cours), vérifie sur le web les indicateurs de référence des églises orientées croissance et complète la SPEC §9 si c'est pertinent.

Attends ma validation avant de coder.
```

## Étape 3 — Répondre aux questions, puis lancer UN lot à la fois

```
Questions répondues : [vos réponses].
Mets à jour la SPEC §13 et CLAUDE.md avec ces réponses.
Lance le lot [N] uniquement, en suivant le plan validé. À la fin : lint, typecheck, tests, Lighthouse si le web est touché,
puis mise à jour de CLAUDE.md §6. Résume ce qui est fait, ce qui reste et les décisions à valider.
```

## Bonnes pratiques
- **Un lot = une conversation.** Tapez `/clear` entre deux lots : CLAUDE.md et la SPEC suffisent à reprendre le contexte, et les réponses restent précises.
- **Mode plan (Shift+Tab)** pour tout nouveau lot : Claude propose avant d'agir.
- **Nouvelle idée en cours de route ?** Ne la donnez pas en vrac. Écrivez-la d'abord dans la SPEC (ou demandez : « ajoute ceci à la SPEC §X »), puis demandez à Claude de l'intégrer au plan.
- **Validez sur l'application réelle** à la fin de chaque lot (web + Android) avant de passer au suivant.
- Gardez la SPEC comme **source de vérité unique** : si une règle change, la SPEC change d'abord.
