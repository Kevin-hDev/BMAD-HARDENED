# Attack Patterns: Credentials and Secrets
> Attack surface spanning the full secret lifecycle — storage at rest, memory residence, and inter-process transit.

---

## 1. Memory Scraping

**Concept:** Any process running as the same OS user can read the heap of another process through OS-provided memory interfaces (`/proc/PID/mem` on Linux, `task_vm_read` on macOS, `ReadProcessMemory` on Windows). Secrets loaded into standard string types reside unencrypted in heap memory for the full lifetime of the process.

**What attackers target:**
- Heap regions where string values are allocated
- Stack frames during function calls that handle credentials
- Swap files and hibernation images (secrets paged out to disk)
- Core dump files written on crash

**Conditions that favor extraction:**
- Secrets stored in standard immutable string types (Python `str`, Java `String`, Go `string`) — the runtime never clears them
- Compiler optimization removing a "dead-store" zero-fill written just before a pointer goes out of scope
- Core dumps enabled (`ulimit -c unlimited` on Linux; Windows Error Reporting enabled)
- Debug symbols present, making pattern matching trivial

**Real incidents:**
- Heartbleed (CVE-2014-0160, April 2014): a single out-of-bounds read of 64 KB from OpenSSL's heap exposed private keys, session tokens, and plaintext passwords from live servers worldwide.
- Any process-level malware on a developer machine can dump API keys from editor/IDE memory — no escalation required.

---

## 2. Environment Variable Leakage

**Concept:** On Linux, `/proc/PID/environ` exposes the full environment block of any process owned by the same user. Environment variables are inherited by every child process spawned with `fork`/`exec`. A secret injected as `API_KEY=...` at process start is readable by any other same-user process for the entire process lifetime.

**What attackers read:**
- `/proc/PID/environ` (Linux) — null-separated name=value pairs
- `/proc/PID/cmdline` (Linux) — secrets passed as CLI arguments, world-readable
- `ps aux` / `wmic process` — command-line arguments visible to all users by default on most systems

**Common variables found:**
`DATABASE_URL`, `AWS_SECRET_ACCESS_KEY`, `GITHUB_TOKEN`, `*_API_KEY`, any secret injected by CI/CD pipelines.

**Cascade risk:** A parent process with a secret in its environment passes it to every child. A compromised child process or a sidecar with write access to its own environment inherits everything.

---

## 3. OS Keystore Extraction

**Concept:** All three major OS keystores (macOS Keychain, Windows Credential Manager via DPAPI, Linux Secret Service) are accessible to any process running as the same user without requiring administrator privileges — by design, since they are meant to serve that user's applications.

**macOS Keychain:** The `security` CLI and Keychain APIs are available to any same-user process. ACLs exist but are only enforced for GUI prompts; code injected into a process that already has access bypasses ACL checks entirely.

**Windows DPAPI / Credential Manager:** `CryptUnprotectData` decrypts blobs using the currently logged-on user's derived key — any process running as that user can call it. Tools like Mimikatz (requires admin) and SharpDPAPI (user-level) automate extraction.

**Linux Secret Service (D-Bus):** The Secret Service API has no per-application isolation. Any process on the same D-Bus session can enumerate and read every secret stored by any application for that user.

**Linux kernel keyring (`keyutils`):** Keys in the user session keyring (`@u`) are readable by any process in the session via `keyctl print <key_id>`.

**Mitigation gap:** OS keystores protect against offline disk theft but not against same-user code execution — the threat model of a compromised dependency or malicious plugin.

---

## 4. Inter-Process Secret Leakage in Transit

**Concept:** When secrets must cross a process boundary (parent to child, service to plugin, backend to agent), the transmission channel is itself an attack surface.

**Attack vectors by channel:**

| Channel | Attack method | Privilege required |
|---|---|---|
| CLI arguments | `ps aux`, `/proc/PID/cmdline` | Same user |
| Environment variables | `/proc/PID/environ` | Same user |
| stdin pipe | `strace -e read`, `/proc/PID/fd/0` | Same user (Linux) |
| Named pipe / Unix socket | Eavesdrop if no `SO_PEERCRED` check | Same user |
| Shared memory | Direct read if permissions open | Same user |
| Unauthenticated IPC | Message injection — no HMAC, no identity check | Network or local |

**Binary replacement (sidecar attack):** If a subprocess binary is resolved from a writable directory, replacing it with a malicious binary causes the parent to send secrets directly to the attacker on the first IPC exchange.

**DLL/dylib hijacking:** A malicious library placed in the same directory as the subprocess is loaded before the legitimate one (Windows DLL search order, macOS `DYLD_INSERT_LIBRARIES`, Linux `LD_PRELOAD`), intercepting memory reads. Documented in the 3CX supply chain attack (2023).

**Audit finding:** Radically Open Security identified 23 issues on Tauri v2, including an extractable encryption key in an IPC channel (`SubtleCrypto extractable: true`, fixed in PR #9327).

---

## 5. Token Theft and Reuse

**Concept:** API tokens, OAuth access tokens, and session tokens are bearer credentials — whoever holds the token is authenticated, regardless of origin. Unlike passwords, tokens are often long-lived or non-expiring.

**Non-expiring token risk:**
- Telegram Bot tokens: never expire until manually revoked. A stolen token grants permanent, full control of the bot and access to all its update history.
- OAuth2 refresh tokens (Reddit, GitHub, etc.): never expire until explicit revocation. Once extracted from storage, they provide indefinite access.
- API keys for LLM providers: no expiration, no IP restriction by default, valid until rotated.

**Extraction locations (priority order for an attacker):**
1. `.env` files (often committed to Git history — `git log --all -- .env`)
2. Process memory (`/proc/PID/mem`, heap dumps)
3. Log files (tokens leaked via debug logging)
4. Config files in predictable locations (`~/.config/`, `AppData/`)
5. OS keystore (same-user accessible)

**Silent verification technique:** An attacker can verify a stolen token with a minimal API call (e.g., a list-models endpoint) that generates no user-visible alert and costs nothing.

**Real incident (2025):** A hardcoded Telegram bot token in client-side JavaScript exposed 397 conversations with personal data. CVSS score 8.1 (High). The breach was silent — Telegram provides no audit log for token usage.

---

## 6. Credential Stuffing and Timing Attacks

**Credential stuffing:** Reusing username/password pairs from other breaches against a target service. Effective when users reuse passwords and when the service has no rate limiting or anomaly detection.

**Timing attack on secret comparison:** A naive equality check (`==`, `===`, `strcmp`) returns early on the first differing byte, leaking information about how many bytes of a guess match the real secret. For a 6-digit OTP with per-byte comparison, the search space reduces from 10^6 to ~60 measurements.

**Real CVE:** RUSTSEC-2022-0018 / CVE-2022-29185 — `totp-rs` < v1.1.0 used non-constant-time comparison for TOTP codes, making them vulnerable to timing oracle attacks.

**Practical exploitability:** The attack requires network noise to be lower than the timing delta per byte (typically 100-500 ns). Over a low-latency connection with 200+ samples per candidate, per-byte leakage becomes measurable.

---

## 7. Secrets at Rest: Files and Configuration

**Plaintext .env files:** Frequently committed to version control by mistake. Even when excluded from the current HEAD, they persist in Git history and are recoverable with `git log --all`.

**Hardcoded secrets in source code:** Regex-scannable patterns (prefixed API keys, connection strings) are trivially found by automated tools. Once in a public repository, secrets must be considered permanently compromised — rotation is the only remedy.

**Unencrypted credential stores:** JSON-based plugin stores, browser extension storage, and application-specific config files often store tokens without encryption, readable by any file-system-level access.

**Deprecated or unaudited vaults:** The tauri-plugin-stronghold was officially deprecated (GitHub issue #7846, IOTA Stronghold repository archived May 2025) with no completed security audit. Any application still depending on it carries an unaudited cryptographic dependency with no future patches.

---

## Summary Risk Matrix

| Attack | Privilege required | Detection difficulty | Impact |
|---|---|---|---|
| Memory scraping | Same user | Low (no network traffic) | Full key extraction |
| `/proc/environ` read | Same user | None | All env secrets |
| OS keystore dump | Same user | Low | All stored secrets |
| IPC eavesdrop (pipe/strace) | Same user | None | Secrets in transit |
| Binary replacement | Write access to app dir | Low | All future secrets |
| Token reuse | Token only | None (no audit log) | Permanent account access |
| Timing attack on OTP | Network access | Very low | Authentication bypass |
| Git history scan | Repository read | None | Historical secrets |
