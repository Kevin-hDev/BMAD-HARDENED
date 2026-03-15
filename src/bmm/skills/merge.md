# Skill : merge

Guide le merge d'une branche vers la branche cible avec résolution assistée des
conflits.

---

## Déclencheur

L'utilisateur dit : "merge", "fusionne la branche", "intègre dans main", ou
équivalent.

---

## Étapes

### 1. Vérifier l'état avant merge

```
git status          # aucun changement non commité toléré
git fetch origin
git log HEAD..origin/<branche-cible> --oneline   # voir les commits entrants
```

Si des changements non commités existent : stopper et demander à l'utilisateur
de commiter ou stasher avant de continuer.

### 2. Choisir la stratégie

| Situation                          | Stratégie recommandée         |
|------------------------------------|-------------------------------|
| Feature branch → main (PR mergée) | `--squash` ou `--no-ff`       |
| Sync branche avec main             | `rebase` (historique propre)  |
| Hotfix urgent                      | `--no-ff` (traçabilité)       |

Proposer la stratégie à l'utilisateur et attendre confirmation.

### 3. Exécuter le merge

```
git checkout <branche-cible>
git merge --no-ff <branche-source>
```
ou
```
git rebase <branche-cible>
```

### 4. Résoudre les conflits (si présents)

Pour chaque fichier en conflit :
1. Afficher le diff du conflit.
2. Expliquer les deux versions (origine + entrant).
3. Proposer une résolution en conservant l'intention des deux côtés.
4. Attendre validation de l'utilisateur avant d'appliquer.
5. Marquer le conflit résolu : `git add <fichier>`.

Ne jamais résoudre un conflit automatiquement sans montrer les options à l'utilisateur.

### 5. Valider après merge

Lancer lint + typecheck + tests (via le skill `fix-errors` si des erreurs
apparaissent) pour confirmer que le merge n'a pas cassé le build.

### 6. Finaliser

```
git merge --continue   # si rebase/merge interactif
git push origin <branche-cible>
```

Afficher le résumé : commits intégrés, fichiers modifiés, conflits résolus.

---

## Règles

- Ne jamais forcer un push (`--force`) sur main/master sans confirmation explicite.
- Ne jamais merger si lint ou tests échouent.
- En cas de doute sur la résolution d'un conflit : demander, ne pas deviner.
- Toujours travailler avec des chemins de fichier absolus dans les commandes shell.
