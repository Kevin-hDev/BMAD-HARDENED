---
name: 'step-02b-confidence'
description: 'ConfidenceChecker — Quality gate pré-implémentation. Scoring graduel avant de coder.'

insertsAfter: 'step-02 (Load project context)'
insertsBefore: 'step-03 (Detect review continuation)'
appliesTo: ['dev-story', 'quick-dev']
---

# Step 2b: ConfidenceChecker — Pre-execution Quality Gate

> **NOTE :** Ce step est un AJOUT au workflow existant.
> Il ne remplace pas le SelfCheck (step-08) ni le HALT sur 3 échecs.
> Protocole complet : `{project-root}/_bmad/bmm/data/confidence-checker.md`

---

## COMPLEXITY DETECTION (AJOUT HARDENED)

Évaluer le niveau L1-L4 de cette story (voir `{project-root}/_bmad/bmm/data/complexity-levels.md`).
Proposer le niveau à l'utilisateur. Adapter la cérémonie du workflow en conséquence.

---

## OBJECTIF

Scorer la confiance de l'agent AVANT de commencer l'implémentation.
Détecter tôt les ambiguïtés, les risques et les stories trop larges.

---

## PRÉREQUIS

Ce step s'exécute APRÈS :
- Lecture complète du fichier story (step-01)
- Chargement du contexte projet (step-02)
- Exploration des fichiers impactés

---

## EXÉCUTION

### 1. Scorer les 6 critères

Pour chaque critère, attribuer un score 0.0–1.0 :

1. **Compréhension des AC** — Les critères d'acceptation sont-ils clairs et non ambigus ?
2. **Connaissance du code impacté** — Les fichiers concernés ont-ils été lus et compris ?
3. **Stratégie de test** — Le plan de test couvre-t-il tous les cas (nominal, erreur, edge) ?
4. **Impact sécurité** — Les implications sécurité ont-elles été identifiées ?
5. **Estimation de complexité** — La story tient-elle dans une session sans découpage ?
6. **Tests existants** — Aucune régression n'est-elle prévisible sur la suite actuelle ?

### 2. Calculer le score moyen

`score_total = moyenne(critère_1 .. critère_6)`

### 3. Appliquer le seuil

| Score | Décision | Action |
|-------|----------|--------|
| >= 0.90 | **PROCEED** | Continuer vers l'implémentation |
| 0.70–0.89 | **CLARIFY** | Lister les doutes, attendre validation humaine |
| < 0.70 | **STOP** | Story à clarifier ou découper — HALT |

---

## ENREGISTREMENT

Écrire le score dans le fichier story, section `Dev Agent Record` :

```
**Confidence Score:** X.X — PROCEED / CLARIFY / STOP
Critères sous 0.9 : [liste des critères avec note]
```

---

## COMPORTEMENT PAR DÉCISION

### PROCEED (>= 0.90)

Afficher :
```
Confidence Score: X.X — PROCEED
Aucun blocage identifié. Démarrage de l'implémentation.
```
Continuer vers step-03 (detect review continuation).

### CLARIFY (0.70–0.89)

Afficher :
```
Confidence Score: X.X — CLARIFY

Doutes identifiés :
- [critère N] : [description du doute]
...

Validation requise avant de continuer. Confirmer ou clarifier ?
```
Attendre réponse utilisateur. Si confirmé : PROCEED. Si ambiguïté persistante : STOP.

### STOP (< 0.70)

Afficher :
```
Confidence Score: X.X — STOP

Blocages identifiés :
- [critère N] : [description du problème]
...

Story à clarifier ou découper avant implémentation.
Suggéré : relancer create-story ou préciser les AC.
```
HALT — ne pas commencer l'implémentation.

---

## NEXT STEP DIRECTIVE

- **PROCEED** : "**NEXT:** Continuer avec step-03 (detect review continuation)"
- **CLARIFY** : Attendre confirmation utilisateur
- **STOP** : HALT — demander clarification ou découpage
