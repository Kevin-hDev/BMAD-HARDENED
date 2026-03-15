# Hierarchical Rule Loading — Chargement progressif du contexte

> Réduction de 70% des tokens de framework en ne chargeant que les règles pertinentes.

## 3 niveaux de règles

- **L1 (constitution)** : TOUJOURS chargé (~200 tokens). `constitution.yaml` = principes non-négociables.
- **L2 (phase)** : chargé selon la phase courante (~300-500 tokens). Règles de PLAN OU IMPLEMENT OU REVIEW — jamais les trois.
- **L3 (domaine)** : chargé selon le contexte (~200-400 tokens). Règles sécurité SI la story touche la sécurité, règles UX SI la story touche l'UI.

## Réduction estimée

- Total chargé : 700-1100 tokens au lieu de 2500-3500 (~70% de réduction)
- Web search : une recherche en phase RESEARCH, résultats mis en cache pour les phases suivantes

## Chargement automatique

L'agent charge le bon niveau au bon moment sans commande manuelle :

1. Lire la phase courante du workflow
2. Lire le domaine de la story (sécurité ? UI ? backend ?)
3. Charger L1 + L2(phase) + L3(domaine) uniquement

## Correspondances phase → L2

| Phase courante | Fichier L2 à charger |
|---|---|
| RESEARCH / ANALYSIS | `data/global-agent-rules.md` section Web Search |
| PLAN | `data/strict-modes.md` section PLAN |
| IMPLEMENT | `data/confidence-checker.md` + `data/model-routing.md` |
| REVIEW | `data/strict-modes.md` section REVIEW |
| SECURITY | `data/security/index.md` → sélection 3-5 fichiers |

## Correspondances domaine → L3

| Domaine détecté | Fichier L3 à charger |
|---|---|
| Sécurité (auth, crypto, tokens) | `data/security/index.md` → fichiers pertinents |
| UI / UX | `data/design/index.md` → fichiers pertinents |
| Architecture | `data/architecture/index.md` → fichiers pertinents |
| Aucun domaine spécifique | Pas de L3 — L1 + L2 suffisent |

## Règle absolue

JAMAIS charger tous les fichiers d'un dossier d'un coup.
Toujours : index → tags → sélection ciblée.
