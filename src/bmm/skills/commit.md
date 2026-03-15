# Skill : commit

Génère et exécute un commit avec un message conventionnel, adapté au build system
détecté dans le contexte de la session.

---

## Déclencheur

L'utilisateur dit : "commit", "commite ça", "crée un commit", ou équivalent.

---

## Étapes

### 1. Vérifier l'état du dépôt
```
git status
git diff --staged
```
Si rien n'est stagé, demander à l'utilisateur ce qu'il veut inclure avant de continuer.

### 2. Lancer les vérifications pré-commit
Selon le build system détecté, lancer lint + typecheck en séquence :
- Si l'une échoue → stopper, afficher l'erreur, NE PAS commiter.
- Si tout passe → continuer.

### 3. Analyser les changements
Lire `git diff --staged` et identifier :
- **Type** : `feat` / `fix` / `refactor` / `docs` / `test` / `chore` / `perf`
- **Scope** (optionnel) : module ou domaine concerné
- **Description** : résumé en impératif, ≤ 72 caractères

### 4. Générer le message

Format strict :
```
<type>(<scope>): <description>

[body optionnel — si changement non trivial]

[footer optionnel — BREAKING CHANGE ou refs #issue]
```

Exemples :
```
feat(auth): add Ed25519 mutual authentication
fix(relay): prevent unbounded connection map growth
docs(prd): update FR42 measurability criteria
```

### 5. Proposer le message à l'utilisateur
Afficher le message généré et demander confirmation (oui / modifier / annuler).

### 6. Exécuter le commit
```
git commit -m "<message validé>"
```

---

## Règles

- Ne jamais inclure de secret, token ou clé dans le message.
- Ne jamais utiliser `--no-verify`.
- Si le hook pre-commit échoue : corriger le problème, re-stager, créer un NOUVEAU commit (jamais `--amend` sur un commit publié).
