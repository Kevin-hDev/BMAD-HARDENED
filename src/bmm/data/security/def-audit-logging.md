# Defense: Tamper-Evident Audit Logging

> Design and implement audit logs that detect tampering, support forensic investigation,
> and comply with regulatory requirements.

---

## Overview

Audit logs serve two security functions: real-time detection (spotting attacks as they
happen) and post-incident forensics (reconstructing what occurred). Both functions require
logs that attackers cannot modify after the fact.

A log that can be deleted or altered is worse than no log — it creates false confidence.

**Design goals:**
- Every security-relevant event is recorded before any side effect occurs.
- No log entry can be modified or deleted without detection.
- Sensitive data (passwords, tokens, keys) is never written to logs.
- Logs survive the compromise of the system they are running on.

**Regulatory anchors:**
- NIST SP 800-92: protection against intentional modification, periodic audit.
- OWASP Logging Cheat Sheet: event categories, exclusion rules.
- ISO/IEC DIS 24970 (AI System Logging, ballot 2025): applies to AI agent systems.
- EU AI Act Article 12 (effective August 2026): mandatory logging for high-risk AI systems,
  minimum 6-month retention.
- GDPR Article 5(1)(e): storage limitation — do not log PII beyond necessity.

---

## Log Format: Structured JSONL

Use one JSON object per line (JSONL / NDJSON). This format is:
- Parseable by all SIEM tools, `jq`, `grep`, log shippers.
- Compressible 5-10x with zstd on structured JSON.
- Verifiable by sequential read without parsing a binary container format.
- Safe to append to without locking concerns (one atomic write per line).

### Required Fields

```json
{
  "timestamp":     "<ISO-8601 UTC, microsecond precision>",
  "sequence":      42,
  "event_type":    "<CATEGORY>_<action>",
  "severity":      "INFO|WARN|ERROR|CRITICAL",
  "source_module": "<module or component name>",
  "action":        "<what the application did>",
  "result":        "success|failure|blocked",
  "details":       { "<structured context, no secrets>" },
  "app_version":   "<semver>",
  "prev_hash":     "sha256:<64 hex chars>",
  "entry_hash":    "sha256:<64 hex chars>"
}
```

**Sequence number:** Monotonically increasing integer starting at 0. Gaps in sequence
indicate deleted entries — even without hash verification.

**Timestamps:** Always UTC. Microsecond precision. Use a monotonic clock source where
available to prevent system clock manipulation from producing false orderings.

### Event Type Categories (OWASP)

| Category | Prefix | Examples |
|----------|--------|---------|
| Authentication | `AUTHN_` | `AUTHN_login_success`, `AUTHN_login_failure`, `AUTHN_lockout` |
| Authorization | `AUTHZ_` | `AUTHZ_access_granted`, `AUTHZ_access_denied` |
| Cryptographic | `CRYPT_` | `CRYPT_decrypt_success`, `CRYPT_key_derived`, `CRYPT_signature_verified` |
| Data access | `DATA_` | `DATA_read`, `DATA_write`, `DATA_delete`, `DATA_export` |
| Configuration | `CONFIG_` | `CONFIG_changed`, `CONFIG_loaded`, `CONFIG_reset` |
| Security events | `SEC_` | `SEC_canary_accessed`, `SEC_debugger_detected`, `SEC_integrity_failure` |
| System | `SYS_` | `SYS_startup`, `SYS_shutdown`, `SYS_crash`, `SYS_update` |
| Privilege | `PRIV_` | `PRIV_escalation_attempt`, `PRIV_sudo_invoked` |

---

## Hash Chaining

### Algorithm

```
entry_hash = HASH(canonical_json(fields_except_entry_hash) || prev_hash)
```

Genesis `prev_hash` = fixed 64-zero constant. Use the same algorithm throughout; prefix
hashes with algorithm identifier (`sha256:`, `blake3:`). Canonical serialization: sort JSON
keys alphabetically; no extra whitespace; exclude `entry_hash` field from input.

**Algorithm choice:** SHA-256 (universal) or BLAKE3 (3-6x faster with SIMD — preferred for
> 10,000 events/second).

**What chaining detects:** entry modification (hash mismatch), deletion (prev_hash gap),
reordering (sequence gap + hash mismatch). Does NOT detect full chain reconstruction by
an attacker who knows the algorithm — requires external anchors.

### External Anchors

Periodically store the current chain head (`sequence:hash`) in a location independent
from the log file: OS keychain, remote append-only endpoint, or immutable timestamping
service (OpenTimestamps, RFC 3161 TSA). Recommended cadence: every 256 entries or 60 seconds.

---

## Digital Signatures on Log Batches

Build a Merkle tree over each epoch (every 256 entries or every minute) and sign the
Merkle root with Ed25519 (~15µs/signature). This adds non-repudiation and detects chain
reconstruction even without external anchors. Signing key stored in OS keychain; public key
published openly. Pattern: Certificate Transparency (Google Trillian, RFC 6962).

---

## What to Log

### Always Log

- Authentication events (success, failure, lockout, multi-factor).
- Authorization decisions (access granted, access denied, privilege check).
- Privilege changes (role assignment, sudo invocation, permission grant).
- Sensitive data access (who accessed what, when — not the data itself).
- Configuration changes (what changed, previous value hash, new value hash).
- Security control events (integrity check pass/fail, debugger detection,
  canary access, library injection detection).
- Application lifecycle (startup, shutdown, crash with exit code, update applied).
- External communication (connection established to which host, not content).

### Never Log

| Data type | Reason |
|-----------|--------|
| Passwords (even hashed) | Log compromise exposes hash for offline cracking |
| API keys, tokens, secrets | Direct credential theft from logs |
| Encryption keys | Same as above |
| Full credit card numbers | PCI-DSS prohibition |
| Full PII (names, emails, SSN) | GDPR / data minimization |
| Session tokens | Enables session hijacking |
| Private key material | Catastrophic |
| Health information | HIPAA / local equivalents |

**Instead:** Log opaque identifiers. Replace `user@example.com` with
`sha256:a1b2c3d4` (truncated hash of the identifier). Replace a token with
`[token: 12 chars, sha256_prefix: ab12cd34]`.

---

## Log Rotation, Shipping, and Retention

**Rotation:** At each rotation boundary (daily or 50-100 MB): record terminal hash, start
new file with genesis entry referencing previous terminal hash (chain continuity), compress
with zstd (5-10x ratio), sign compressed segment with Ed25519, apply OS immutability
(`chattr +i` on Linux), update segment manifest.

**Remote shipping:** Background worker drains a bounded, persisted queue to a remote
append-only endpoint using separate credentials. Exponential backoff on failure. A shipping
failure lasting > N minutes is itself a P1 alert.

**Retention:**
- Hot (uncompressed JSONL + index): 90 days — instant query.
- Warm (compressed + signed): 12 months — decompress on demand.
- Cold (signed archives): 3-7 years — manual restore.

Regulatory minimums: EU AI Act Article 12 = 6 months; CNIL = 6-12 months. GDPR Article
5(1)(e): do not retain longer than necessary.

---

## Code Review Checklist

- [ ] Log files opened in append-only mode (no seek, no truncate).
- [ ] Every entry contains `sequence`, `prev_hash`, `entry_hash`.
- [ ] Hash computed over canonically serialized content (deterministic).
- [ ] Genesis entry uses documented fixed `prev_hash`.
- [ ] No secrets, passwords, tokens, PII written to log content.
- [ ] Sensitive identifiers replaced with opaque hashes.
- [ ] Event types follow the defined category naming scheme.
- [ ] Timestamps are UTC, microsecond precision.
- [ ] Verification function exists: reads all entries, recomputes all hashes, reports chain breaks.
- [ ] External anchors stored (keychain and/or remote endpoint).
- [ ] Log rotation preserves chain continuity via terminal hash reference in next genesis.
- [ ] Rotated segments signed and marked immutable.
- [ ] Segment manifest maintained.
- [ ] Remote shipping uses separate credentials.
- [ ] Shipping queue is bounded and persisted.
- [ ] Retention policy documented and enforced.
- [ ] No `catch {}` or silent failures around log write operations — log write failure
  must be treated as a critical error, not silently ignored.
