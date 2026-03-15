# Détection automatique du build system

## Instructions pour l'agent dev

Au démarrage de toute session de développement, **scanner la racine du projet** pour
détecter l'écosystème. Sauvegarder le résultat dans le contexte de la session sous
la clé `build_system`. Si plusieurs marqueurs coexistent (monorepo), détecter chaque
sous-système séparément (voir section Monorepo).

---

## Fichiers marqueurs par écosystème

| Fichier marqueur     | Écosystème      |
|----------------------|-----------------|
| `Cargo.toml`         | Rust            |
| `package.json`       | Node / JS / TS  |
| `pyproject.toml`     | Python (modern) |
| `requirements.txt`   | Python (legacy) |
| `go.mod`             | Go              |
| `build.gradle`       | Java / Kotlin   |
| `build.gradle.kts`   | Kotlin          |
| `Gemfile`            | Ruby            |
| `pubspec.yaml`       | Flutter / Dart  |
| `*.csproj`           | .NET / C#       |
| `CMakeLists.txt`     | C / C++         |
| `Package.swift`      | Swift           |

---

## Commandes par écosystème

### Rust
```
build : cargo build
test  : cargo test
lint  : cargo clippy -- -D warnings
fmt   : cargo fmt --check
```

### Node / JS / TS
```
build : npm run build  (ou yarn build / pnpm build)
test  : npm test
lint  : npm run lint
fmt   : npm run format
```
> Détecter le package manager : présence de `yarn.lock` → yarn, `pnpm-lock.yaml` → pnpm, sinon npm.

### Python
```
build : python -m build
test  : pytest
lint  : ruff check .
fmt   : ruff format --check .
```

### Go
```
build : go build ./...
test  : go test ./...
lint  : golangci-lint run
fmt   : gofmt -l .
```

### Java / Kotlin (Gradle)
```
build : ./gradlew build
test  : ./gradlew test
lint  : ./gradlew detekt  (Kotlin) / checkstyle (Java)
fmt   : ./gradlew ktlintFormat
```

### Ruby
```
build : bundle install
test  : bundle exec rspec
lint  : bundle exec rubocop
fmt   : bundle exec rubocop -a
```

### Flutter / Dart
```
build : flutter build <target>
test  : flutter test
lint  : flutter analyze
fmt   : dart format --set-exit-if-changed .
```

### .NET / C#
```
build : dotnet build
test  : dotnet test
lint  : dotnet format --verify-no-changes
fmt   : dotnet format
```

### C / C++ (CMake)
```
build : cmake --build build/
test  : ctest --test-dir build/
lint  : clang-tidy <files>
fmt   : clang-format --dry-run --Werror <files>
```

### Swift
```
build : swift build
test  : swift test
lint  : swiftlint lint
fmt   : swiftformat --lint .
```

---

## Support monorepo

Si plusieurs fichiers marqueurs sont détectés dans des sous-répertoires différents :

1. Scanner récursivement jusqu'à la profondeur 3.
2. Construire une map `{ chemin_sous_repertoire → écosystème }`.
3. Appliquer les commandes de chaque sous-système indépendamment lors des étapes
   build/test/lint.
4. Exemple de contexte sauvegardé :
   ```
   build_system:
     backend/: rust
     frontend/: node
     mobile/: flutter
   ```

---

## Algorithme de décision

1. Lire les fichiers à la racine.
2. Vérifier chaque marqueur dans l'ordre du tableau ci-dessus.
3. Premier marqueur trouvé → écosystème principal.
4. Si aucun marqueur → demander à l'utilisateur.
5. Stocker le résultat dans le contexte avant toute action de build.
