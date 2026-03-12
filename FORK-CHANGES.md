# BMAD-HARDENED — Changes from original project

> Fork of [BMad Method v6.0.4](https://github.com/bmad-code-org/BMAD-METHOD) by Brian (BMad) Madison / BMad Code, LLC.
> All changes below are additions or reinforcements. No original code was removed.

---

## New agents (3)

| File | Agent | Description |
|------|-------|-------------|
| `src/bmm/agents/cybersecurity-expert.agent.yaml` | Nyx (🛡️) | Cybersecurity expert — vulnerabilities, OWASP, CVE, supply chain, LLM security |
| `src/bmm/agents/security-architect.agent.yaml` | Bastion (🏰) | Security architect — threat modeling STRIDE/DREAD, zero-trust, crypto design |
| `src/bmm/agents/geek.agent.yaml` | Zero (🤓) | Tech genius — bleeding-edge watch, lesser-known alternatives, hacker mindset |

## New workflow (1)

| File | Description |
|------|-------------|
| `src/bmm/workflows/3-solutioning/security-review/workflow.md` | Security Review — comprehensive security audit of PRD, architecture, and stories before implementation |

## New security DATA files (25)

Directory: `src/bmm/data/security/`

### Attack patterns (11)

| File | Content |
|------|---------|
| `atk-llm-prompt-injection.md` | Direct/indirect prompt injection, jailbreak, extraction |
| `atk-llm-deception.md` | Scheming, behavioral manipulation, LLM deception |
| `atk-injection-input.md` | SQL/NoSQL/LDAP/XPath injection, XSS, template injection |
| `atk-command-injection.md` | OS command injection, path traversal, SSRF |
| `atk-supply-chain.md` | Typosquatting, dependency confusion, build pipeline |
| `atk-credentials-secrets.md` | Credential stuffing, token theft, secret exposure |
| `atk-network-tls.md` | MITM, certificate pinning bypass, protocol downgrade |
| `atk-privilege-escalation.md` | Local/network elevation, container escape, RBAC bypass |
| `atk-ipc-process.md` | IPC hijacking, shared memory, named pipe attacks |
| `atk-chain-patterns.md` | Multi-step kill chains, lateral movement |
| `atk-reverse-engineering.md` | Binary analysis, deobfuscation, anti-tamper bypass |

### Defense patterns (10)

| File | Content |
|------|---------|
| `def-llm-pipeline.md` | Secure LLM pipeline, prompt sandboxing, output filtering |
| `def-crypto-secrets.md` | Modern crypto choices, secrets management, key rotation |
| `def-auth-patterns.md` | Auth patterns (OAuth2, mTLS, Ed25519), session management |
| `def-network-tls.md` | TLS 1.3, certificate pinning, network isolation |
| `def-os-isolation.md` | OS sandboxing, containers, seccomp, AppArmor |
| `def-ipc-hardening.md` | Secure IPC, message validation, serialization |
| `def-framework-hardening.md` | Framework hardening, CSP, CORS, headers |
| `def-runtime-memory.md` | Memory zeroization, runtime protection, anti-debug |
| `def-deception-monitoring.md` | Honeypots, canary tokens, anomaly detection |
| `def-audit-logging.md` | Secure audit logging, tamper-proof, compliance |

### Reference (3)

| File | Content |
|------|---------|
| `ref-agent-threat-model.md` | Threat model template for AI agents |
| `ref-cross-validation-matrix.md` | Security cross-validation matrix |
| `ref-cve-catalog.md` | CVE reference catalog by category |

### Index

| File | Content |
|------|---------|
| `index.md` | Tag-based index for selective loading (INDEX_THEN_SELECTIVE) |

## Modified existing agents (10)

| Agent | Changes |
|-------|---------|
| PM (product-manager) | + web search for market/trends, + FR verification against checkup-feature.md, + global rules |
| SM (scrum-master) | + web search, + story size enforcement (max 5 AC / 3 tasks / 5-7 files), + global rules |
| QA (quality-assurance) | + web search for test frameworks, + security-focused test cases, + global rules |
| Architect | + web search, + global rules |
| Developer (dev) | + web search, + global rules |
| Analyst | + web search, + global rules |
| Tech Writer | + web search principle, + global rules *(added in 6.1.1)* |
| UX Designer | + web search principle, + global rules *(added in 6.1.1)* |
| Cybersecurity Expert (Nyx) | + global rules *(added in 6.1.1)* |
| Security Architect (Bastion) | + global rules *(added in 6.1.1)* |
| Tech Genius (Zero) | + global rules *(added in 6.1.1)* |

## Modified existing workflows (19)

| Workflow | Changes |
|----------|---------|
| `code-review/workflow.md` | + adversarial security deep dive, + INDEX_THEN_SELECTIVE security data loading, - file size check (redundant with CLAUDE.md) |
| `security-review/workflow.md` | Updated to INDEX_THEN_SELECTIVE |
| `create-architecture/workflow.md` | + security_data path, + security data loading section |
| `create-story/workflow.md` | + security_data in Input Files, + security enrichment in step 3 |
| `create-epics-and-stories/workflow.md` | + security_data path, + section for security ACs |
| `dev-story/workflow.md` | + conditional security data loading (when story has security ACs) |
| `check-implementation-readiness/workflow.md` | + security_data path, + BLOCKING security gate |
| **All 19 BMM workflows** | + `PRE-EXECUTION (MANDATORY)` block — loads global-agent-rules.md + requires web search before any step *(added in 6.1.2)* |

## Modified existing step files (96)

All 96 step files across all workflows now include a web search instruction in their `MANDATORY EXECUTION RULES` section. This ensures agents search the web at every step, even after context compaction or in new conversations. *(added in 6.1.3)*

## Modified/added global files (2)

| File | Changes |
|------|---------|
| `src/bmm/data/global-agent-rules.md` | + Web Search Systematic, + Checkup Feature Tracking, + Story Size Enforcement, + Code Review Scope, + Security DATA Loading (INDEX_THEN_SELECTIVE) |
| `src/bmm/templates/checkup-feature.md` | New feature tracking template across phases |

---

## Web search enforcement (3-layer)

BMAD-HARDENED enforces web search at **three levels** to survive context compaction:

1. **Agent YAML** (principles) — loaded at conversation start
2. **Workflow PRE-EXECUTION** — loaded when workflow launches
3. **Step MANDATORY EXECUTION RULES** — loaded at each step, survives compaction

This redundancy is intentional. LLMs lose early instructions during long workflows.

---

## Philosophy

- **Universal**: All security files are language/framework agnostic. They describe patterns, not specific implementations.
- **Non-invasive**: No original code was removed. All changes are additions or reinforcements.
- **Selective**: Data is never loaded in bulk — the INDEX_THEN_SELECTIVE mechanism ensures minimal, relevant loading.
