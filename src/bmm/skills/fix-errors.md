# Skill : fix-errors

Détecte et corrige les erreurs de lint, typecheck et compilation en parallèle via
sous-agents, puis valide que tout passe avant de continuer.

---

## Déclencheur

L'utilisateur dit : "fixe les erreurs", "corrige le lint", "répare les warnings",
ou ce skill est appelé automatiquement en pré-condition d'un commit ou d'une PR.

---

## Étapes

### 1. Détecter le build system

Lire `build_system` depuis le contexte de session (voir `build-detection.md`).
Si absent, relancer la détection.

### 2. Lancer les checks en parallèle (sous-agents)

Lancer simultanément :
- **Sous-agent lint** : exécuter la commande lint de l'écosystème, capturer la sortie.
- **Sous-agent typecheck** : exécuter le typecheck (si applicable), capturer la sortie.
- **Sous-agent test** (optionnel, si demandé) : exécuter les tests unitaires.

Exemples pour Rust :
```
lint      : cargo clippy -- -D warnings
typecheck : cargo check
```

Exemples pour Node/TS :
```
lint      : npm run lint
typecheck : npx tsc --noEmit
```

### 3. Analyser les erreurs

Pour chaque erreur retournée :
- Identifier le fichier et la ligne.
- Classifier : lint / type / compilation / test.
- Grouper les erreurs similaires pour les corriger en une seule passe.

### 4. Corriger par ordre de priorité

1. Erreurs de compilation (bloquantes).
2. Erreurs de typecheck.
3. Erreurs de lint (auto-fixables en priorité).

Appliquer les corrections fichier par fichier. Ne jamais modifier plus de 250 lignes
par fichier en une seule passe.

### 5. Relancer la validation

Après corrections, relancer lint + typecheck pour confirmer que tout passe.
Si des erreurs persistent, recommencer les étapes 3–5 (maximum 3 itérations).

### 6. Rapport final

Afficher :
- Nombre d'erreurs corrigées.
- Fichiers modifiés.
- Éventuelles erreurs résiduelles nécessitant une intervention manuelle.

---

## Règles

- Ne jamais ignorer une erreur avec un commentaire `// eslint-disable` ou `#[allow(...)]`
  sans l'accord explicite de l'utilisateur.
- Ne jamais supprimer du code pour faire disparaître une erreur.
- Si une erreur est ambiguë, demander avant de corriger.
