# Strict Mode Enforcement — BMAD v2

> Chaque agent DOIT déclarer son mode avant chaque action.
> Ce fichier est chargé au démarrage de tout workflow.

---

## Déclaration obligatoire

Chaque réponse d'agent DOIT commencer par la déclaration de mode :

```
[MODE: RESEARCH] | [MODE: PLAN] | [MODE: IMPLEMENT] | [MODE: REVIEW] | [MODE: SECURITY]
```

Aucune action n'est valide sans déclaration préalable.

---

## Permissions par mode

### [MODE: RESEARCH]

| | Détail |
|---|---|
| **AUTORISÉ** | Lire des fichiers, web search, explorer le codebase, lancer des sous-agents d'exploration |
| **INTERDIT** | Créer ou modifier des fichiers source, écrire du code d'implémentation |

### [MODE: PLAN]

| | Détail |
|---|---|
| **AUTORISÉ** | Écrire des specs, stories, plans, AC, architecture docs |
| **INTERDIT** | Écrire du code d'implémentation |

### [MODE: IMPLEMENT]

| | Détail |
|---|---|
| **AUTORISÉ** | Écrire du code, des tests, modifier des fichiers source |
| **INTERDIT** | Modifier des specs approuvées, changer le plan en cours |

### [MODE: REVIEW]

| | Détail |
|---|---|
| **AUTORISÉ** | Lire et commenter du code, exécuter tests/lint |
| **INTERDIT** | Écrire du nouveau code, modifier des fichiers source |

### [MODE: SECURITY]

| | Détail |
|---|---|
| **AUTORISÉ** | Auditer la sécurité, écrire des recommandations, charger les DATA files |
| **INTERDIT** | Modifier du code — recommandations seulement, jamais d'implémentation directe |

---

## Transitions de mode

Le changement de mode est tracé dans le fichier story.
Chaque transition nécessite :

1. **Déclaration explicite** — `[MODE: X → Y]` dans la réponse
2. **Vérification des outputs** — les outputs du mode précédent doivent être présents
3. **Traçabilité** — la transition est notée dans la section `## Progress` de la story

### Séquence normale

```
RESEARCH → PLAN → IMPLEMENT → REVIEW → SECURITY
```

### Transitions valides (non-séquentielles)

- `IMPLEMENT → SECURITY` — découverte d'un problème de sécurité en cours d'implémentation
- `REVIEW → IMPLEMENT` — fix d'un bug mineur découvert en review
- `PLAN → RESEARCH` — gap de connaissance découvert pendant la planification

### Transitions invalides

- `RESEARCH → IMPLEMENT` — on ne code jamais sans plan
- `IMPLEMENT → PLAN` — on ne replanifie pas pendant l'implémentation (créer une nouvelle story)

---

## Violation de mode

Si un agent tente une action interdite dans son mode actuel :

1. **STOP** — l'action est bloquée
2. **LOG** — noter la tentative dans la story : `[VIOLATION: action X tentée en MODE: Y]`
3. **ESCALADE** — si violation répétée, escalader au PM

---

## Référence rapide

| Mode | Peut lire | Peut écrire code | Peut écrire docs | Peut auditer |
|------|-----------|-----------------|-----------------|--------------|
| RESEARCH | OUI | NON | NON | NON |
| PLAN | OUI | NON | OUI | NON |
| IMPLEMENT | OUI | OUI | NON | NON |
| REVIEW | OUI | NON | NON | NON |
| SECURITY | OUI | NON | OUI (recommandations) | OUI |
