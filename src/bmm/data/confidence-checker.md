---
title: ConfidenceChecker — Pre-execution Quality Gate
type: protocol
version: 1.0
---

# ConfidenceChecker — Pre-execution Quality Gate

> AJOUT au SelfCheck existant (quick-dev step-04, dev-story step-08).
> Ce protocole intervient AVANT l'implémentation, pas après.
> Référence seuils et template : `shared/meta/confidence-checker.md`

## Quand l'exécuter

- AVANT chaque story implementation :
  - dev-story : entre step-02 (contexte chargé) et step-05 (implémentation)
  - quick-dev : entre step-02 (context-gathering) et step-03 (execute)
- APRÈS avoir lu les AC et exploré le code existant

## 6 critères de scoring (0.0–1.0 chacun)

| # | Critère | Question de contrôle |
|---|---------|---------------------|
| 1 | Compréhension des AC | Les critères d'acceptation sont-ils clairs et non ambigus ? |
| 2 | Connaissance du code impacté | Les fichiers concernés ont-ils été lus et compris ? |
| 3 | Stratégie de test | Le plan de test couvre-t-il tous les cas (nominal, erreur, edge) ? |
| 4 | Impact sécurité | Les implications sécurité ont-elles été identifiées ? |
| 5 | Estimation de complexité | La story tient-elle dans une seule session sans découpage ? |
| 6 | Tests existants | Aucune régression n'est-elle prévisible sur la suite actuelle ? |

## Seuils de décision

| Score moyen | Action |
|-------------|--------|
| >= 0.90 | **PROCEED** — Continuer l'implémentation |
| 0.70–0.89 | **CLARIFY** — Lister les doutes, demander validation humaine |
| < 0.70 | **STOP** — Story à clarifier ou découper avant implémentation |

## Template de scoring

```
## Confidence Score

| Critère | Score | Note |
|---------|-------|------|
| 1. Compréhension AC | X.X | [raison si < 0.9] |
| 2. Code impacté connu | X.X | [raison si < 0.9] |
| 3. Stratégie de test | X.X | [raison si < 0.9] |
| 4. Impact sécurité | X.X | [raison si < 0.9] |
| 5. Complexité estimée | X.X | [raison si < 0.9] |
| 6. Régression risque | X.X | [raison si < 0.9] |

**Score total : X.X/1.0**
**Décision : PROCEED / CLARIFY / STOP**
**Doutes bloquants :** [liste si CLARIFY ou STOP]
```

## Enregistrement

Le score est écrit dans le fichier story sous la section `Dev Agent Record`.
Format : `Confidence Score: X.X — PROCEED/CLARIFY/STOP`
