# Skill : create-pr

Crée une Pull Request avec un titre et une description générés depuis les Stories
associées au travail en cours.

---

## Déclencheur

L'utilisateur dit : "crée une PR", "ouvre une pull request", ou équivalent.

---

## Prérequis

- Branche courante ≠ branche principale (main/master).
- Les commits à inclure sont prêts et poussés (ou on pousse dans cette étape).

---

## Étapes

### 1. Collecter le contexte

```
git log main..HEAD --oneline
git diff main...HEAD --stat
```

Identifier les Stories couvertes par les commits (numéros dans les messages,
ou via le contexte de session).

### 2. Pousser la branche si nécessaire

```
git push -u origin <branche-courante>
```

### 3. Générer le contenu de la PR

**Titre** (≤ 70 caractères) :
```
<type>(<scope>): <résumé de l'ensemble des changements>
```

**Body** (template) :
```markdown
## Résumé

- <bullet point par Story ou groupe de changements majeurs>

## Stories couvertes

- Story <ID> — <titre>
- ...

## Plan de test

- [ ] <vérification manuelle ou test automatisé>
- [ ] Lint et typecheck passent
- [ ] Pas de régression sur les tests existants

## Notes pour le reviewer

<informations contextuelles utiles, décisions prises>
```

### 4. Proposer à l'utilisateur

Afficher titre + body. Permettre correction avant création.

### 5. Créer la PR

```
gh pr create --title "<titre>" --body "<body>"
```

Retourner l'URL de la PR créée.

---

## Règles

- Ne jamais créer de PR depuis main vers main.
- Ne jamais inclure de secret ou donnée sensible dans la description.
- Si `gh` n'est pas disponible, fournir le titre + body à copier-coller manuellement.
