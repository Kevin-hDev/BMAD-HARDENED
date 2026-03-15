# Niveaux de complexité — Adaptation automatique de la cérémonie

> Le workflow s'adapte à la taille de la tâche. Moins de cérémonie pour une typo, plus pour un système.

## Détection au démarrage

Évaluer la complexité de la story/tâche :

### L1 — Trivial (~500 tokens de framework)
**Critères :** fix typo, renommage, changement de config, < 3 fichiers.

Étapes à sauter :
- Plan détaillé
- Architecture review
- Design review

Étapes conservées :
- Confidence check rapide (score unique, pas de détail)
- Implémentation
- Validate
- Security check léger (scan surface uniquement)

### L2 — Simple (~1500 tokens de framework)
**Critères :** bug fix, petit feature isolé, 3-7 fichiers.

Étapes à sauter :
- Architecture review
- Design review

Étapes conservées :
- Plan court (bullet points)
- Implémentation
- Validate
- Security check standard

### L3 — Feature (~3000 tokens de framework)
**Critères :** nouvelle fonctionnalité, 7-15 fichiers, impact sur plusieurs couches.

→ Workflow BMAD complet, aucun step supprimé.

### L4 — Système (~5000 tokens de framework)
**Critères :** refactoring majeur, > 15 fichiers, changement d'architecture ou de contrat d'API.

→ Workflow complet + étapes supplémentaires :
- Architecture review obligatoire
- Security review approfondie
- Five-Persona active

## Override utilisateur

L'utilisateur peut forcer un niveau avec le flag `--level N` (ex : `--level 3`).

Sans override, le système **propose automatiquement** le niveau détecté et **demande confirmation** avant de continuer.

Format de proposition :
```
Complexité détectée : L2 (simple) — bug fix, 4 fichiers estimés.
Workflow adapté : plan court + implement + validate + security standard.
[C] Confirmer  [+] Monter d'un niveau  [-] Descendre d'un niveau
```

## Impact par workflow

| Workflow | Niveaux applicables | Adaptation |
|---|---|---|
| `dev-story` | L1-L4 | Skip steps selon niveau (voir ci-dessus) |
| `quick-dev` | L1-L2 uniquement | Rôle natif du workflow — escalade si L3+ |
| `security-review` | L3 = standard, L4 = Five-Persona active | |
| `code-review` | L1 = review légère, L4 = review approfondie multi-persona | |

## Règle d'escalade

Si `quick-dev` détecte L3 ou L4 → escalader automatiquement vers le Full BMad Flow.
Afficher : "Cette story dépasse le périmètre de quick-dev (L3+). Recommandation : utiliser dev-story."
