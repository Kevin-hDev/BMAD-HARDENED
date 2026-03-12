# BMAD-HARDENED — Modifications par rapport au projet original

> Fork de [BMad Method v6.0.4](https://github.com/bmad-code-org/BMAD-METHOD) par Brian (BMad) Madison / BMad Code, LLC.
> Toutes les modifications ci-dessous sont des ajouts ou renforcements. Le code original n'a pas été supprimé.

---

## Nouveaux agents (3)

| Fichier | Agent | Description |
|---------|-------|-------------|
| `src/bmm/agents/cybersecurity-expert.agent.yaml` | Nyx (🛡️) | Expert cybersécurité — vulnérabilités, OWASP, CVE, supply chain, sécurité LLM |
| `src/bmm/agents/security-architect.agent.yaml` | Bastion (🏰) | Architecte sécurité — threat modeling STRIDE/DREAD, zero-trust, crypto design |
| `src/bmm/agents/geek.agent.yaml` | Zero (🤓) | Tech Genius — veille bleeding-edge, alternatives méconnues, hacker mindset |

## Nouveau workflow (1)

| Fichier | Description |
|---------|-------------|
| `src/bmm/workflows/3-solutioning/security-review/workflow.md` | Security Review — audit sécurité complet du PRD, architecture et stories avant implémentation |

## Nouveaux fichiers de DATA sécurité (25)

Répertoire : `src/bmm/data/security/`

### Patterns d'attaque (11)

| Fichier | Contenu |
|---------|---------|
| `atk-llm-prompt-injection.md` | Injection de prompt directe/indirecte, jailbreak, extraction |
| `atk-llm-deception.md` | Scheming, manipulation comportementale, déception LLM |
| `atk-injection-input.md` | SQL/NoSQL/LDAP/XPath injection, XSS, template injection |
| `atk-command-injection.md` | OS command injection, path traversal, SSRF |
| `atk-supply-chain.md` | Typosquatting, dependency confusion, build pipeline |
| `atk-credentials-secrets.md` | Credential stuffing, token theft, secret exposure |
| `atk-network-tls.md` | MITM, certificate pinning bypass, protocol downgrade |
| `atk-privilege-escalation.md` | Elevation locale/réseau, container escape, RBAC bypass |
| `atk-ipc-process.md` | IPC hijacking, shared memory, named pipe attacks |
| `atk-chain-patterns.md` | Kill chains multi-étapes, lateral movement |
| `atk-reverse-engineering.md` | Binary analysis, deobfuscation, anti-tamper bypass |

### Patterns de défense (10)

| Fichier | Contenu |
|---------|---------|
| `def-llm-pipeline.md` | Pipeline LLM sécurisé, sandboxing prompt, output filtering |
| `def-crypto-secrets.md` | Choix crypto modernes, gestion secrets, key rotation |
| `def-auth-patterns.md` | Auth patterns (OAuth2, mTLS, Ed25519), session management |
| `def-network-tls.md` | TLS 1.3, certificate pinning, network isolation |
| `def-os-isolation.md` | Sandboxing OS, containers, seccomp, AppArmor |
| `def-ipc-hardening.md` | IPC sécurisé, validation messages, serialization |
| `def-framework-hardening.md` | Hardening par framework, CSP, CORS, headers |
| `def-runtime-memory.md` | Zeroization mémoire, protection runtime, anti-debug |
| `def-deception-monitoring.md` | Honeypots, canary tokens, anomaly detection |
| `def-audit-logging.md` | Audit logging sécurisé, tamper-proof, compliance |

### Référence (3)

| Fichier | Contenu |
|---------|---------|
| `ref-agent-threat-model.md` | Template de threat model pour agents AI |
| `ref-cross-validation-matrix.md` | Matrice de cross-validation sécurité |
| `ref-cve-catalog.md` | Catalogue CVE de référence par catégorie |

### Index

| Fichier | Contenu |
|---------|---------|
| `index.md` | Index avec tags pour chargement sélectif (INDEX_THEN_SELECTIVE) |

## Agents existants modifiés (7)

| Agent | Modifications |
|-------|--------------|
| PM (product-manager) | + recherche web marché/tendances, + vérification FR contre checkup-feature.md, + global rules |
| SM (scrum-master) | + recherche web, + story size enforcement (max 5 AC / 3 tasks / 5-7 files), + global rules |
| QA (quality-assurance) | + recherche web frameworks test, + test cases sécurité, + global rules |
| Architect | + recherche web, + global rules |
| Developer (dev) | + recherche web, + global rules |
| Analyst | + recherche web, + global rules |
| Tech Writer | + recherche web, + global rules |

## Workflows existants modifiés (6)

| Workflow | Modifications |
|----------|--------------|
| `code-review/workflow.md` | + deep dive sécurité adversarial, + chargement data sécurité INDEX_THEN_SELECTIVE, - file size check (redondant CLAUDE.md) |
| `security-review/workflow.md` | Mise à jour vers INDEX_THEN_SELECTIVE |
| `create-architecture/workflow.md` | + path security_data, + section chargement data sécurité |
| `create-story/workflow.md` | + security_data dans Input Files, + enrichissement sécurité en step 3 |
| `create-epics-and-stories/workflow.md` | + path security_data, + section pour ACs sécurité |
| `dev-story/workflow.md` | + chargement conditionnel data sécurité (si story a des ACs sécurité) |
| `check-implementation-readiness/workflow.md` | + path security_data, + gate sécurité BLOQUANTE |

## Fichiers globaux modifiés/ajoutés (2)

| Fichier | Modifications |
|---------|--------------|
| `src/bmm/data/global-agent-rules.md` | + Web Search Systematic, + Checkup Feature Tracking, + Story Size Enforcement, + Code Review Scope, + Security DATA Loading (INDEX_THEN_SELECTIVE) |
| `src/bmm/templates/checkup-feature.md` | Nouveau template de suivi des features par phase |

---

## Philosophie

- **Universel** : Tous les fichiers sécurité sont agnostiques en langage/framework. Ils décrivent des patterns, pas des implémentations spécifiques.
- **Non invasif** : Le code original n'a pas été supprimé. Les modifications sont des ajouts ou des renforcements.
- **Sélectif** : Les données ne sont jamais chargées en bloc — le mécanisme INDEX_THEN_SELECTIVE garantit un chargement minimal et pertinent.
