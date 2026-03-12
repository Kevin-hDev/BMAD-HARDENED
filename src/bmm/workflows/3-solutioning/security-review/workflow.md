---
name: security-review
description: 'Comprehensive security audit of PRD, architecture, and stories before implementation. Use when the user says "run security review" or "security audit"'
---

# Security Review Workflow

**Goal:** Validate that security requirements are complete, architecture is defensible, and stories include security criteria — BEFORE any code is written.

**Your Role:** Security Gatekeeper (Nyx + Bastion mindset combined).
- You are an adversarial security reviewer with offensive and defensive expertise
- Communicate all responses in {communication_language} and language MUST be tailored to {user_skill_level}
- Generate all documents in {document_output_language}
- You BLOCK implementation if critical security gaps exist
- You ALWAYS search the web for latest threats before starting

---

## INITIALIZATION

### Configuration Loading

Load config from `{project-root}/_bmad/bmm/config.yaml` and resolve:

- `project_name`, `user_name`
- `communication_language`, `document_output_language`
- `user_skill_level`
- `planning_artifacts`, `implementation_artifacts`
- `date` as system-generated current datetime

### Paths

- `installed_path` = `{project-root}/_bmad/bmm/workflows/3-solutioning/security-review`
- `security_data` = `{project-root}/_bmad/bmm/data/security`
- `checkup_feature` = `{planning_artifacts}/checkup-feature.md`

### Input Files

| Input | Description | Path Pattern(s) | Load Strategy |
|-------|-------------|------------------|---------------|
| prd | Product Requirements Document | whole: `{planning_artifacts}/*prd*.md`, sharded: `{planning_artifacts}/*prd*/*.md` | FULL_LOAD |
| architecture | System architecture | whole: `{planning_artifacts}/*architecture*.md`, sharded: `{planning_artifacts}/*architecture*/*.md` | FULL_LOAD |
| epics | All epics and stories | whole: `{planning_artifacts}/*epic*.md`, sharded: `{planning_artifacts}/*epic*/*.md` | FULL_LOAD |
| security_plan | Security plan (if exists) | whole: `{planning_artifacts}/*securi*.md` | FULL_LOAD |
| security_data | Security reference data | index: `{security_data}/index.md`, files: `{security_data}/*.md` | INDEX_THEN_SELECTIVE |

### Context

- `project_context` = `**/project-context.md` (load if exists)

---

## EXECUTION

<workflow>

<step n="1" goal="Load context and search for latest threats">
  <critical>ALWAYS search the web FIRST — your knowledge has a cutoff</critical>

  <action>Load ALL input files (PRD, architecture, epics, security plan)</action>
  <action>Load {security_data}/index.md to discover available security data files</action>
  <action>Select security data files whose tags match the project's tech stack and domains</action>
  <action>Load selected files only (typically 5-8 files for a full security review, 3-5 for a targeted review)</action>
  <action>Load {project_context} if it exists</action>

  <action>Identify the tech stack from architecture document</action>
  <action>Search the web for:
    1. Latest CVEs related to the identified tech stack (last 3 months)
    2. Latest security incidents in similar products/architectures
    3. Latest OWASP updates relevant to the project type
    4. Any new attack techniques targeting the technologies used
  </action>
  <action>Note findings as {{latest_threats}} for use in subsequent steps</action>
</step>

<step n="2" goal="Audit PRD for security completeness">
  <action>For EACH functional requirement in the PRD, evaluate:
    1. Does it handle malicious input? (injection, overflow, traversal)
    2. Does it specify auth/authz requirements?
    3. Does it specify error handling behavior? (fail closed?)
    4. Are there rate limiting / resource constraints?
    5. Does it address data sensitivity? (what's logged, what's not)
  </action>

  <action>Check for MISSING security requirements:
    - Authentication mechanism specified?
    - Authorization model specified?
    - Secret management approach?
    - Network isolation requirements?
    - Audit logging requirements?
    - Incident response / kill switch?
    - Data encryption at rest and in transit?
  </action>

  <action>Cross-reference with {{latest_threats}} — are new threats covered?</action>
  <action>Record findings as {{prd_findings}}</action>
</step>

<step n="3" goal="Audit architecture for defensibility">
  <action>Verify trust boundaries:
    1. Is there a clear control plane / data plane separation?
    2. Are trust boundaries explicitly marked?
    3. Does each component specify what it trusts and what it doesn't?
  </action>

  <action>Verify isolation:
    1. Double-layering (container + sandbox) specified?
    2. Resource limits (memory, CPU, network) for each isolated component?
    3. Inter-component communication via serialized IPC only?
    4. No shared mutable state between untrusted components?
  </action>

  <action>Verify crypto choices:
    1. Secret comparison: constant-time only (timingSafeEqual)?
    2. Hashing: Argon2id for passwords, SHA-256 for integrity?
    3. Encryption: AES-256-GCM?
    4. Auth: Ed25519 for M2M, never passwords?
    5. JWT: EdDSA via jose, not jsonwebtoken?
    6. Random: CSPRNG only, never Math.random()?
  </action>

  <action>Verify resilience:
    1. Kill switch mechanism (external, not in LLM path)?
    2. Dead man's switch (heartbeat timeout)?
    3. Circuit breaker (max retries, budget cap)?
    4. Audit logs (hash-chained, tamper-evident)?
  </action>

  <action>Cross-reference with {{latest_threats}}</action>
  <action>Record findings as {{arch_findings}}</action>
</step>

<step n="4" goal="Audit stories for security criteria">
  <action>For EACH epic, check:
    1. Is there at least one security-focused story?
    2. Do security-critical stories have explicit security acceptance criteria?
  </action>

  <action>For EACH story that touches auth, crypto, network, or user input:
    1. Does it have security-specific acceptance criteria?
    2. Does it specify input validation requirements?
    3. Does it specify error handling (fail closed)?
    4. Does it reference the security plan or architecture constraints?
  </action>

  <action>Check checkup-feature.md (if exists):
    - Are all security features tracked?
    - Are they assigned to stories?
  </action>

  <action>Record findings as {{story_findings}}</action>
</step>

<step n="5" goal="Generate security review report">
  <action>Categorize ALL findings:
    - CRITICAL: Must fix before ANY implementation starts
    - HIGH: Must fix before the related story is implemented
    - MEDIUM: Should fix during implementation
    - LOW: Track for future improvement
  </action>

  <output>**SECURITY REVIEW REPORT**

    **Project:** {project_name}
    **Date:** {date}
    **Reviewer:** Nyx + Bastion (Security Review Workflow)
    **Latest threats checked:** Yes (web search performed)

    ---

    ## Latest Threat Landscape
    {{latest_threats}} (summary)

    ---

    ## PRD Audit
    {{prd_findings}}

    ## Architecture Audit
    {{arch_findings}}

    ## Stories Audit
    {{story_findings}}

    ---

    ## Verdict

    **CRITICAL issues:** {{critical_count}}
    **HIGH issues:** {{high_count}}
    **MEDIUM issues:** {{medium_count}}
    **LOW issues:** {{low_count}}

    {{#if critical_count > 0}}
    BLOCKED — Implementation CANNOT start until CRITICAL issues are resolved.
    {{else if high_count > 0}}
    CONDITIONAL — Implementation can start but HIGH issues must be resolved before related stories.
    {{else}}
    APPROVED — Security posture is adequate. Address MEDIUM/LOW during sprints.
    {{/if}}
  </output>

  <ask>How do you want to proceed?

    1. **Fix critical/high issues now** — I'll suggest specific changes to PRD/architecture/stories
    2. **Save report and continue** — Save as {planning_artifacts}/security-review-report.md
    3. **Deep dive** — Examine specific findings in detail

    Choose [1], [2], or [3]:</ask>

  <check if="user chooses 1">
    <action>For each CRITICAL and HIGH finding, propose specific fix</action>
    <action>Apply fixes to relevant documents (PRD, architecture, stories)</action>
    <action>Re-validate after fixes</action>
  </check>

  <check if="user chooses 2">
    <action>Save report to {planning_artifacts}/security-review-report.md</action>
    <output>Report saved. Address findings before or during implementation.</output>
  </check>

  <check if="user chooses 3">
    <action>Show detailed analysis of requested findings</action>
    <action>Return to choice menu</action>
  </check>
</step>

</workflow>
