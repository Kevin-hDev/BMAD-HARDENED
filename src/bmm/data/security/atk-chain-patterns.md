# Attack Chain Patterns

> Multi-step attack sequences that pivot across components to maximize impact.

---

## Overview

Single-component vulnerabilities rarely achieve high impact. Attackers chain vulnerabilities
across components, using each compromise as a pivot point to the next. Understanding these
chains is essential for designing defenses at the right boundaries.

**Key principle:** An XSS alone is low severity. XSS that invokes a native IPC call, which
spawns a subprocess with injected arguments, which escalates privileges — that is full system
compromise. Defense must account for the full chain, not isolated vulnerabilities.

---

## Common Pivot Points

| From | To | Pivot Mechanism |
|------|-----|-----------------|
| Frontend (WebView/browser) | Native IPC | Malicious JS calls native bridge |
| Native IPC | Filesystem | IPC command reads/writes arbitrary paths |
| Native IPC | Subprocess | IPC spawns external process with attacker-controlled args |
| Subprocess | OS privilege | Subprocess calls privileged helper (pkexec, sudo, UAC) |
| External content (RSS, scraping) | Frontend | Content rendered without sanitization triggers XSS |
| LLM pipeline | Tool execution | Injected prompt triggers agentic tool calls |
| Supply chain dep | Runtime | Malicious package executes at build or import time |
| CI/CD secrets | Signing key | Exposed private key signs malicious update |
| Memory dump | Credential | Process memory contains plaintext secrets |

---

## Chain 1: Content Injection → IPC → Subprocess → Privilege Escalation

**Entry point:** Application renders external content (RSS feed, web scraping, markdown).

**Stages:**
1. Attacker controls external content consumed by the application.
2. Content contains a payload that executes in the rendering context (XSS, script injection).
3. The injected script calls the application's native IPC bridge with attacker-controlled arguments.
4. The IPC command spawns a subprocess or shell with those arguments unsanitized (`shell=True` pattern).
5. The subprocess invokes a privileged helper — escalating to root or SYSTEM.

**Conditions required:** Content rendered without sanitization; IPC allows arbitrary arguments
(`"args": true` in capability declarations); subprocess uses shell interpolation; privileged
helper reachable without strong auth (polkit, UAC default level).

**Impact:** Full system compromise (root/SYSTEM) from reading a malicious article.

**Detection points:**
- Unexpected native IPC calls immediately after rendering external content.
- Subprocess arguments containing shell metacharacters (`;`, `|`, `&&`, `$(...)`).
- Privileged helper invocations not triggered by user action.

---

## Chain 2: Supply Chain Compromise → Build Injection → Runtime Exfiltration

**Entry point:** A dependency (npm package, Cargo crate, PyPI package) is compromised.

**Stages:**
1. Attacker compromises a popular dependency (phishing maintainer, typosquatting, account takeover).
2. Malicious code is injected into the package and included in the next build.
3. The injected code installs runtime hooks (intercepting network calls, filesystem reads).
4. At runtime, secrets (API keys, tokens) passing through those hooks are silently exfiltrated.

**Real incident:** September 2025 — chalk and debug npm packages (2.6 billion weekly downloads)
compromised for 18 packages; CISA alert September 23, 2025.

**Conditions required:** No dependency integrity verification (lockfile hash checks, `cargo deny`,
`npm audit signatures`); secrets passed through network layer interceptable by injected code.

**Impact:** Exfiltration of all API keys and tokens processed by the application.

**Detection points:**
- Unexpected outbound network calls to unknown domains during application startup.
- Dependency hash mismatch between expected and installed versions.
- `sendBeacon`, `XMLHttpRequest` or `fetch` calls in dependency source not present in audit.

---

## Chain 3: Prompt Injection → LLM Tool Call → Command Execution

**Entry point:** External content is processed by an LLM with tool access.

**Stages:**
1. Attacker embeds hidden instructions in content the LLM will process (invisible text in HTML,
   metadata fields, document comments — cross-prompt injection attack / XPIA).
2. The LLM receives the poisoned content and follows the injected instructions.
3. In multi-agent pipelines, the output of a compromised agent is passed as trusted context
   to the next agent, propagating the injection (auto-replication).
4. The LLM uses an available tool (bash, filesystem, network) to execute the attacker's command.

**Research reference:** ICLR 2025 — 65.2% propagation success rate in multi-agent chains.
HiddenLayer April 2025 (Policy Puppetry) — universal bypass on GPT-4o, Claude 3.5/3.7, Gemini 2.5.

**Conditions required:** LLM has access to side-effecting tools; external content processed
without content isolation; multi-agent pipeline treats agent output as trusted input.

**Impact:** Arbitrary code execution via the LLM's tool invocations.

**Detection points:**
- LLM tool calls not correlating with any explicit user request.
- Unusual arguments to "safe" tools (`find`, `grep`, `git`) that include network exfiltration patterns.
- Instructions embedded in unexpected fields (metadata, comments, invisible elements).

---

## Chain 4: Memory Access → Credential Extraction → Persistent C2

**Entry point:** Attacker has user-level access to the same OS session as the target process.

**Stages:**
1. Process memory is read via OS facilities (`/proc/PID/mem` on Linux, `procdump` on Windows)
   without requiring a debugger or elevated privileges — same-user access is sufficient.
2. Secrets stored in plaintext strings in heap memory are extracted via pattern matching.
3. Extracted tokens (API keys, bot tokens, OAuth tokens) are used to establish a persistent
   command-and-control channel.
4. If the extracted token is used for alerting (e.g., a Telegram bot token), the attacker
   hijacks the alerting channel itself — intercepting all security alerts before the defender
   sees them, and optionally sending fake alerts or silence.

**Conditions required:** Secrets stored in unprotected string memory (not zeroed after use);
same-user process access; token has no rotation, no secondary auth, no expiry.

**Impact:** Silent persistent access; neutralization of the alerting system.

**Detection points:**
- Unexpected processes reading `/proc/PID/mem` or calling `OpenProcess` with `PROCESS_VM_READ`.
- Alert delivery suddenly stops while suspicious activity continues.
- Webhook configuration changes on communication channels.

---

## Chain 5: CI/CD Compromise → Signed Malicious Update → Mass Persistence

**Entry point:** The build and release pipeline is compromised.

**Stages:**
1. A GitHub Actions action (or equivalent) is compromised via a mutable tag reference (`@v4`
   instead of pinned SHA) — CVE-2025-30066 (tj-actions/changed-files, CVSS Critical, 2025).
2. The compromised action leaks CI secrets (code signing private key) into build logs.
3. Attacker creates a backdoored binary and signs it with the stolen private key.
4. The signature is valid — the application's auto-updater accepts and installs it on all
   existing user instances simultaneously.

**Conditions required:** Actions pinned by mutable tag; signing key stored as CI secret
accessible to workflow steps; auto-update mechanism without secondary verification.

**Impact:** Persistent backdoor deployed to all users of the application in one operation.

**Detection points:**
- CI logs containing secret patterns (base64 blobs, key headers).
- New release not correlating with any commit or PR in source history.
- Binary hash mismatch between what CI produced and what the release contains.

---

## Chain 6: Reverse Engineering → Binary Replacement → Persistent Backdoor

**Entry point:** Attacker has read access to an installed application binary.

**Stages:**
1. The installed binary (or a bundled interpreted component) is extracted and decompiled.
2. Source code, hardcoded secrets, and internal logic are recovered.
3. A modified version of the binary is created that forwards all traffic to the legitimate
   binary while silently exfiltrating data.
4. The replacement binary is installed in place of the original.
5. Without integrity verification before execution, the backdoored binary runs on every
   application launch.

**Conditions required:** No pre-execution integrity check (SHA-256 comparison) on critical
binaries; binary is extractable (PyInstaller, Electron ASAR, etc.); no OS-level signature
verification at load time.

**Impact:** All application secrets exfiltrated on every launch; code execution under
application identity.

**Detection points:**
- Binary hash mismatch at startup (if verification is implemented).
- Unexpected network connections from the application process.
- File modification timestamp on binary not matching the last known update.

---

## Lateral Movement and Kill Chain Reference

**Common lateral movement after initial access:**
- Keyring sweep: Linux Secret Service has no per-app isolation — any same-user process reads all stored secrets.
- Environment harvest: `/proc/PID/environ` readable by same-user without privilege.
- `.ssh/` exfiltration: keys without passphrase enable access to all SSH-connected servers.
- Shared writable `site-packages` / `node_modules`: one compromised app affects all.

**MITRE ATT&CK references:** T1195 (Supply Chain), T1059 (Scripting Interpreter), T1055 (Process Injection), T1555 (Credential Stores), T1567 (Exfiltration Over Web Service), T1562 (Impair Defenses), T1202 (Indirect Command Execution).
