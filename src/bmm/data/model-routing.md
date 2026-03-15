---
title: Model Routing — Suggestion automatique
type: protocol
version: 1.0
---

# Model Routing — Suggestion automatique

> Ce fichier définit le routing suggéré par type de tâche.
> L'utilisateur peut toujours overrider la suggestion.
> Règle absolue : JAMAIS Opus en sous-agent (sauf demande explicite).

## Table de routing par type de tâche

| Type de tâche | Modèle suggéré | Raison |
|--------------|----------------|--------|
| Exploration, recherche, résumé, lecture de code | **Haiku** | Rapide, économique, lecture seule |
| Fix typo, renommage, changement trivial | **Haiku** | Complexité trop faible pour Sonnet |
| Écriture de code, debug, tests, refactor | **Sonnet** | Bon rapport qualité/coût pour le code |
| Security review, threat modeling | **Sonnet** | Analyse sécurité structurée |
| Architecture complexe, review critique, conception | **Opus** | Raisonnement profond nécessaire |
| Story standard (1-5 fichiers) | **Sonnet** | Référence par défaut |

## Règle absolue

**JAMAIS Opus en sous-agent** — Opus est réservé à la conversation principale uniquement,
sauf si l'utilisateur en fait la demande explicite.

## Affichage au démarrage du workflow

À chaque début de workflow impliquant du code, l'agent SUGGÈRE le modèle :

```
Modèle recommandé : [Haiku / Sonnet / Opus]
Raison : [type de tâche + nombre de fichiers estimé]
Override : répondre avec le modèle souhaité pour changer.
```

L'utilisateur peut ignorer ou overrider sans justification.

## Critères de décision rapide

- **1-3 fichiers, lecture seule** → Haiku
- **1-5 fichiers, écriture de code** → Sonnet
- **Raisonnement cross-domaine, architecture système** → Opus
- **10+ fichiers avec analyse globale** → sous-agent Sonnet (jamais Opus)

## Chargement

Ce fichier est référencé dans `global-agent-rules.md`.
Les agents chargent ce fichier lors de l'initialisation du workflow.
