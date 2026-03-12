# Defense: Deception and Active Monitoring

> Deploy tripwires, canaries, and tamper-evident monitoring to detect unauthorized access.

---

## Overview

Deception-based defenses operate on a different principle than prevention: instead of
blocking attackers, they make attackers detectable. Every time an attacker touches a
tripwire, you get high-confidence evidence of compromise — not a false-positive alert,
but a confirmed intrusion event.

**Core properties of effective deception:**
- Canary names must be indistinguishable from real sensitive files.
- Access to any canary is a high-confidence signal (near-zero false positive rate).
- Alert delivery must be independent of the monitored system (attacker cannot silence it).
- Logs must be tamper-evident so attackers cannot erase their traces.

**Absolute naming rule:** Canary file names must NEVER contain: `canary`, `trap`, `honey`,
`fake`, `decoy`, `bait`, `lure`, `test`, `dummy`, `sample`. Names must be realistic and
indistinguishable from genuine sensitive files.

---

## Canary Files

**Placement:** Application config dirs, `~/.config/`, `~/.ssh/`, Documents, Desktop —
locations attackers systematically enumerate.

**Naming — high-attractiveness examples** (all targeted by TruffleHog, GitGuardian, gitleaks):

`.env.production`, `credentials.json`, `private_key.pem`, `service_account.json`,
`master.key`, `backup.db`, `recovery_codes.txt`, `.pgpass`, `token.json`, `wallet.dat`,
`admin_creds.xlsx`, `config.bak`, `.npmrc`, `shadow.bak`, `api_keys.json`.

**Content:** Structurally valid but non-functional credentials (use documented example formats:
AWS example keys, Stripe test key format, etc.). For open-source projects, generate canary
file paths at deployment from a per-installation random seed — never store names in source.

### File Read Detection by Platform

| Platform | Mechanism | Notes |
|----------|-----------|-------|
| Linux | inotify `IN_OPEN \| IN_ACCESS` | No privilege required; standard `notify` libraries only watch writes — use raw inotify API |
| Linux (elevated) | fanotify | Provides accessor PID; can block file opens; requires CAP_SYS_ADMIN |
| Windows | Poll `File.accessed()` every 5s | Enable: `fsutil behavior set disablelastaccess 0`; 5s TOCTOU window |
| macOS | Canary tokens (HTTP beacon) | Endpoint Security Framework requires Apple entitlement; not available to standard apps |

---

## Canary Tokens and Honeypots

**Canary tokens** are non-functional credentials embedded in config, databases, or API
responses. They trigger an alert when used — even by a remote attacker with stolen data.

- **Database record canaries:** Sentinel rows in real tables. Alert on: read, login attempt,
  deletion (deletion = database tampering confirmed).
- **Config file tokens:** Fake credential in distributed config. If it appears in any request,
  that config was exfiltrated.
- **API response injection:** Fake key returned for specific synthetic queries. Appearance in
  any subsequent request = client compromised.

**Honeypot endpoints:** Services/ports/routes with no legitimate use. Any access = confirmed
hostile event. Examples: `/admin` with no nav link, unused DB account, closed port that logs
connection attempts.

**Alert immediately** on any access. Do not attempt to authenticate or serve content —
log everything about the request (headers, source, payload) and alert.

---

## Hash-Chained Tamper-Evident Logging

### Purpose

Standard log files can be edited after the fact. An attacker who gains system access
can delete log lines recording their activity. Hash-chained logs make tampering
detectable: any modification, deletion, or reordering of entries breaks the chain.

### Structure

Each log entry contains:
- The hash of the previous entry (`prev_hash`).
- A hash of the current entry's content computed after `prev_hash` is set (`entry_hash`).

Format (one JSON object per line — JSONL/NDJSON):
```
{
  "timestamp": "<ISO-8601 with microseconds, UTC>",
  "sequence": <monotonically increasing integer>,
  "event_type": "<category>_<action>",
  "severity": "INFO|WARN|ERROR|CRITICAL",
  "action": "<what happened>",
  "result": "<outcome>",
  "details": { ... },
  "prev_hash": "sha256:<hex> or blake3:<hex>",
  "entry_hash": "sha256:<hex> or blake3:<hex>"
}
```

**Genesis entry:** First entry uses a fixed `prev_hash` of 64 zeros (documented constant).

**Hash algorithm:** SHA-256 (universally available) or BLAKE3 (3-6x faster with SIMD,
recommended for high-volume logging). Use the same algorithm consistently throughout the log.

**Canonical serialization:** Hash must be computed over a deterministic serialization.
Sort JSON keys alphabetically before hashing, or serialize fields in a fixed defined order.

### Strengthening Against Reconstruction Attacks

An attacker with knowledge of the algorithm can reconstruct a valid chain after deleting
entries (recalculating all hashes). Mitigation: **periodic external anchors**.

Store the current chain head hash (sequence number + hash) in a location independent
from the log file:
- OS keychain/keystore (attacker who can modify logs may not have keychain access).
- Remote log shipping endpoint (see below).
- Periodic hash publication to an immutable external record.

During verification, compare chain head hashes against stored anchors. A reconstructed
chain cannot match anchors it did not know about at time of reconstruction.

**Append-only enforcement:** Linux: `chattr +a` (prevents truncation; requires root to set);
`chattr +i` on finalized rotated segments. Windows: open with `FILE_APPEND_DATA` only.
Application level: `O_APPEND` flag always set.

**Log rotation integrity:** On rotation — record terminal hash; new file genesis references
previous terminal hash (cross-file chain continuity); compress with zstd; sign with Ed25519;
store `.sig` adjacent; mark rotated segment immutable.

**Remote shipping:** Background worker drains bounded persisted queue to append-only remote
endpoint using separate credentials. Shipping delivery failure is a P1 alert.

---

## Behavioral Anomaly Detection

| Behavior | Signal |
|----------|--------|
| N auth failures in T seconds | Brute force — apply exponential backoff tarpit |
| Multiple canary files accessed | Automated credential scanner active |
| Unexpected PID reading own config | Library injection hook |
| Network to unknown destination | Exfiltration or C2 beacon |
| Binary hash mismatch at startup | Binary replaced on disk |
| Log chain break | Log tampering attempted |
| Alert delivery failure (extended) | Alert channel sabotaged |

**Tarpit:** Exponential backoff per identifier: 0s, 2s, 4s, 8s, 16s... capped at 60s;
blacklist for 1 hour after 10 failures. State map size-bounded (max 10,000 entries with LRU
eviction) to prevent memory exhaustion.

---

## Alert Escalation

| Priority | Events | Delivery |
|----------|--------|---------|
| P0 Critical | Canary accessed, debugger detected, binary tampered, canary record deleted | < 1 second, loud |
| P1 Warning | Log chain break, config changed, 3+ auth failures/5 min, unknown library | Batch 5 min |
| P2 Info | Startup, integrity check pass, log rotation | Daily digest, silent |

**Storm protection:** 5+ P1 events in 10 minutes → aggregate to single P0 "alert storm" +
30-minute cooldown (individual P0 events still pass through).

**Alert channel independence:** Alert credentials separate from application credentials.
Alert queue persisted to disk. Delivery failure is itself monitored (P1 alert).

---

## Checklist

- [ ] Canary files deployed with realistic names (no forbidden words).
- [ ] Canary file paths randomized at deployment, not hardcoded in source.
- [ ] Canary file contents are structurally valid but non-functional credentials.
- [ ] File read monitoring active (platform-appropriate mechanism).
- [ ] Canary database records present and monitored for deletion or access.
- [ ] Honeypot endpoints configured and alerting.
- [ ] Audit log uses JSONL format with hash chaining.
- [ ] Hash chain verification tool available and tested.
- [ ] Periodic chain head anchored externally (keychain or remote).
- [ ] Log files opened append-only at OS level.
- [ ] Tarpit active on authentication with bounded state map.
- [ ] Alert escalation matrix defined (P0/P1/P2).
- [ ] Alert channel is independent of monitored system.
- [ ] Alert delivery failure is itself monitored and alerted.
- [ ] Alert storm protection configured.
