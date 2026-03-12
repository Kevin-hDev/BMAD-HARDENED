# BMAD-HARDENED

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Based on BMad Method](https://img.shields.io/badge/Based%20on-BMad%20Method%20v6-blue)](https://github.com/bmad-code-org/BMAD-METHOD)

**Fork communautaire de la [BMad Method](https://github.com/bmad-code-org/BMAD-METHOD)** avec un renforcement sécurité intégré dans tout le cycle de vie agile.

> **Credit** : Ce projet est un fork de **[BMad Method](https://github.com/bmad-code-org/BMAD-METHOD)** par **Brian (BMad) Madison / [BMad Code, LLC](https://github.com/bmad-code-org)**. Tout le framework de base, l'architecture des agents, les workflows et le CLI sont son travail. BMAD-HARDENED ajoute une couche sécurité par-dessus cette fondation.

---

## Ce que BMAD-HARDENED ajoute

### 3 nouveaux agents sécurité (Party Mode)

| Agent | Nom | Role |
|-------|------|------|
| 🛡️ Cybersecurity Expert | **Nyx** | Analyse de vulnérabilités, OWASP, CVE tracking, supply chain, sécurité LLM |
| 🏰 Security Architect | **Bastion** | Threat modeling STRIDE/DREAD, zero-trust, isolation, crypto design |
| 🤓 Tech Genius | **Zero** | Veille techno bleeding-edge, alternatives aux solutions mainstream |

### 1 nouveau workflow : Security Review

Audit de sécurité complet de l'architecture, du PRD et des stories **avant** l'implémentation. Intégré dans la phase 3 (Solutioning).

### 24 fichiers de DATA sécurité universels

Base de connaissances sécurité chargeable **à la demande** par n'importe quel agent/workflow via un système d'index par tags :

- **11 patterns d'attaque** (`atk-*`) : injection LLM, supply chain, privilege escalation, reverse engineering...
- **10 patterns de défense** (`def-*`) : crypto, auth, isolation OS, hardening framework, audit logging...
- **3 fichiers de référence** (`ref-*`) : threat model agent, matrice de cross-validation, catalogue CVE

Tous les fichiers sont **agnostiques** en langage/framework — ils décrivent des patterns universels.

### Renforcement des agents et workflows existants

- **Tous les agents** : recherche web systématique avant chaque travail + respect des global rules
- **Code Review** : deep dive sécurité adversarial ajouté au workflow
- **5 workflows** enrichis avec chargement conditionnel des data sécurité (INDEX_THEN_SELECTIVE)
- **Global Agent Rules** : web search obligatoire, checkup feature, story size enforcement, review scope guard
- **Implementation Readiness** : gate sécurité bloquante avant le passage en Phase 4

### Stratégie de chargement : INDEX_THEN_SELECTIVE

Les données sécurité ne sont **jamais** chargées en bloc. Le mécanisme :
1. L'agent/workflow charge `index.md` (listing des fichiers avec tags)
2. Il match les tags avec le contexte actuel (stack, domaine, story)
3. Il charge **seulement 3-5 fichiers** pertinents

3 couches de défense garantissent le chargement :
1. Global rules → référencent index.md
2. Chaque workflow → instructions explicites de chargement
3. Critical actions des agents → référencent index.md

---

## Installation

```bash
npx bmad-method install
```

> Ce fork est compatible avec le CLI standard de BMad Method. Installez-le normalement, puis remplacez les fichiers du module `bmm` par ceux de ce fork.

---

## Projet original

**[BMad Method](https://github.com/bmad-code-org/BMAD-METHOD)** par Brian (BMad) Madison / BMad Code, LLC

- [Documentation officielle](https://docs.bmad-method.org)
- [Discord communautaire](https://discord.gg/gk8jAdXWmj)
- [Soutenir BMad](https://buymeacoffee.com/bmad)

---

## Changelog

Voir [FORK-CHANGES.md](FORK-CHANGES.md) pour la liste complète des modifications apportées par ce fork.

Voir [CHANGELOG.md](CHANGELOG.md) pour l'historique du projet original.

## License

MIT License — voir [LICENSE](LICENSE).

---

**BMad** et **BMAD-METHOD** sont des marques de BMad Code, LLC. Voir [TRADEMARK.md](TRADEMARK.md).
Ce fork est un projet communautaire non officiel, non affilié ni approuvé par BMad Code, LLC.
