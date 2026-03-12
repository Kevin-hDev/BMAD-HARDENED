# Reference: Attack-to-Defense Cross-Validation Matrix

> Map identified vulnerabilities to defensive controls; expose coverage gaps.

---

## Purpose

This matrix answers: "For every attack pattern identified, do we have a corresponding defense?"

It prevents the failure mode where attacks and defenses are documented separately with no
systematic verification that they correspond.

**Outcome of a complete review:**
- Every attack has a defensive control mapped to it.
- Every gap is explicitly acknowledged with severity and remediation plan.
- Coverage metrics give an objective view of security posture.

---

## Process (5 Steps)

1. **Enumerate attacks** — Using threat modeling or attack simulation, list all identified
   vulnerabilities. Assign each a unique ID (`ATK-001`, etc.).

2. **Map to defenses** — For each `ATK-ID`, find the corresponding defensive control.
   Record the defense file and section.

3. **Assign coverage status:**
   - `covered` — Defense fully documented with implementation guidance.
   - `partial` — Defense exists but is incomplete or has known gaps.
   - `gap` — No defense documented for this attack.

4. **Classify gaps by severity:**
   - `P0` — CVSS >= 9.0, or direct path to full system compromise. Block release.
   - `P1` — CVSS 7.0-8.9, or significant impact. Requires remediation plan with deadline.
   - `P2` — Medium or low severity. Acceptable with documented risk justification.

5. **Calculate and report:**
   ```
   coverage_rate = (count "covered") / (total attacks)
   target: >= 80% covered, 0 P0 gaps
   ```

**A P0 gap in the final report = review failed = do not ship.**

---

## Acceptance Thresholds

| Attack severity | Required coverage | Action on gap |
|-----------------|-------------------|---------------|
| P0 Critical (CVSS >= 9.0) | 100% covered | Block release; fix immediately |
| P1 High (CVSS 7.0-8.9) | >= 90% covered | Remediation plan + deadline required |
| P2 Medium/Low | >= 70% covered | Documented risk acceptance acceptable |

---

## Pre-Populated Coverage Table

Use this as a starting point. Add project-specific rows. Status: `covered` / `partial` / `gap`.

| ATK-ID | Attack | Sev | Status | Key Controls | Defense Ref |
|--------|--------|-----|--------|-------------|------------|
| ATK-INJ-001 | Command injection via shell interpolation | P0 | | No shell=True; args as list; regex allowlist | atk-chain-patterns.md Chain 1 |
| ATK-INJ-002 | SQL injection via dynamic query construction | P0 | | Parameterized queries only; no string concat | |
| ATK-INJ-003 | XSS in browser/WebView | P0 | | HTML sanitization; CSP header | |
| ATK-PI-001 | XPIA in LLM pipeline | P0 | | Sanitize before LLM; spotlighting; output scanning | atk-chain-patterns.md Chain 3 |
| ATK-PI-002 | LLM agentic tool call with attacker args | P0 | | Schema validation; human approval for high-risk tools | |
| ATK-RE-001 | Binary analysis reveals source/secrets | P1 | | Symbols stripped; LTO; no hardcoded secrets | atk-reverse-engineering.md |
| ATK-RE-002 | Debug attachment extracts runtime secrets | P1 | | Anti-debug in production; secrets zeroed after use; no core dumps | |
| ATK-SC-001 | Malicious dependency via package registry | P0 | | Lockfile verified; audit in CI; Actions pinned by SHA | atk-chain-patterns.md Chain 2 |
| ATK-AUTH-001 | Credential brute force | P0 | | Exponential backoff; lockout; bounded tarpit state | def-deception-monitoring.md |
| ATK-AUTH-002 | Timing attack on secret comparison | P0 | | Constant-time comparison only; no == on secrets | |
| ATK-MON-001 | Audit log tampered or deleted | P0 | | Hash-chained JSONL; external anchor; append-only | def-audit-logging.md |
| ATK-MON-002 | Alert channel hijacked | P0 | | Alert creds separate; webhook secret token; delivery failure monitored | def-deception-monitoring.md |
| ATK-PRIV-001 | Library injection (LD_PRELOAD / DLL / DYLD) | P1 | | LD_PRELOAD cleared at startup; Hardened Runtime (macOS) | atk-reverse-engineering.md |

---

## Gap Analysis Template

```yaml
identified_gaps:
  - gap_id: "GAP-001"
    description: ""
    severity: "P0 | P1 | P2"
    atk_id: ""
    recommendation: ""
    effort: "low | medium | high"
    deadline: ""
    accepted_risk_reason: ""   # P2 only
```

---

## Summary Report Template

```yaml
cross_validation_summary:
  review_date: "<YYYY-MM-DD>"
  product_version: "<version>"
  coverage_statistics:
    total_attacks: 0
    covered: 0
    partial: 0
    gap: 0
    coverage_rate: "0%"
  p0_gaps: []        # Must be empty — any entry here = FAIL
  p1_gaps: []        # Each must have remediation plan + deadline
  p2_gaps: []        # Each must have risk acceptance justification
  validation_result: "PASS | FAIL"
  # FAIL: any P0 gap OR coverage_rate < 80%
```
