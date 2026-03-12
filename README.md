# BMAD-HARDENED

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Based on BMad Method](https://img.shields.io/badge/Based%20on-BMad%20Method%20v6-blue)](https://github.com/bmad-code-org/BMAD-METHOD)

**Community fork of [BMad Method](https://github.com/bmad-code-org/BMAD-METHOD)** with integrated security hardening across the entire agile lifecycle.

> **Credit**: This project is a fork of **[BMad Method](https://github.com/bmad-code-org/BMAD-METHOD)** by **Brian (BMad) Madison / [BMad Code, LLC](https://github.com/bmad-code-org)**. The entire base framework, agent architecture, workflows, and CLI are his work. BMAD-HARDENED adds a security layer on top of this foundation.

---

## What BMAD-HARDENED adds

### 3 new security agents (Party Mode)

| Agent | Name | Role |
|-------|------|------|
| 🛡️ Cybersecurity Expert | **Nyx** | Vulnerability analysis, OWASP, CVE tracking, supply chain, LLM security |
| 🏰 Security Architect | **Bastion** | Threat modeling STRIDE/DREAD, zero-trust, isolation, crypto design |
| 🤓 Tech Genius | **Zero** | Bleeding-edge tech watch, alternatives to mainstream solutions |

### 1 new workflow: Security Review

Comprehensive security audit of architecture, PRD, and stories **before** implementation. Integrated in Phase 3 (Solutioning).

### 24 universal security DATA files

Security knowledge base loaded **on demand** by any agent/workflow via a tag-based index system:

- **11 attack patterns** (`atk-*`): LLM injection, supply chain, privilege escalation, reverse engineering...
- **10 defense patterns** (`def-*`): crypto, auth, OS isolation, framework hardening, audit logging...
- **3 reference files** (`ref-*`): agent threat model, cross-validation matrix, CVE catalog

All files are **language/framework agnostic** — they describe universal patterns, not specific implementations.

### Reinforcement of existing agents and workflows

- **All agents**: systematic web search before any work + global rules compliance
- **Code Review**: adversarial security deep dive added to workflow
- **5 workflows** enriched with conditional security data loading (INDEX_THEN_SELECTIVE)
- **Global Agent Rules**: mandatory web search, feature checkup tracking, story size enforcement, review scope guard
- **Implementation Readiness**: blocking security gate before Phase 4

### Loading strategy: INDEX_THEN_SELECTIVE

Security data is **never** loaded in bulk. The mechanism:
1. Agent/workflow loads `index.md` (file listing with tags)
2. Matches tags against current context (stack, domain, story)
3. Loads **only 3-5 relevant files**

3 defense layers ensure loading:
1. Global rules reference index.md
2. Each workflow has explicit loading instructions
3. Agent critical_actions reference index.md

---

## Installation

```bash
npx bmad-method install
```

> This fork is compatible with the standard BMad Method CLI. Install normally, then replace the `bmm` module files with those from this fork.

---

## Original project

**[BMad Method](https://github.com/bmad-code-org/BMAD-METHOD)** by Brian (BMad) Madison / BMad Code, LLC

- [Official documentation](https://docs.bmad-method.org)
- [Community Discord](https://discord.gg/gk8jAdXWmj)
- [Support BMad](https://buymeacoffee.com/bmad)

---

## Changelog

See [FORK-CHANGES.md](FORK-CHANGES.md) for the complete list of modifications made by this fork.

See [CHANGELOG.md](CHANGELOG.md) for the original project history.

## License

MIT License — see [LICENSE](LICENSE).

---

**BMad** and **BMAD-METHOD** are trademarks of BMad Code, LLC. See [TRADEMARK.md](TRADEMARK.md).
This fork is an unofficial community project, not affiliated with or endorsed by BMad Code, LLC.
