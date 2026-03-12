# Attack Patterns: Inter-Process Communication (IPC)
> Attack surface, techniques, and code review indicators for IPC bridges in desktop, mobile, and server applications.

---

## Overview

IPC bridges connect privileged backend processes to less-trusted frontend or UI layers. Any JavaScript execution, malformed message, or misconfigured permission model can allow an attacker to invoke privileged operations, escalate privileges, or exfiltrate data. The attack surface is present in all desktop frameworks (Electron, Tauri, NW.js, Flutter desktop), mobile IPC (Android Binder, iOS XPC), and server sidecar architectures.

---

## ATK-IPC-01 — Direct IPC Invocation Bypassing UI

**Description:** The IPC bridge is exposed to the frontend layer. An attacker with JavaScript execution can call any backend command authorized by the current permission scope, bypassing UI-level controls entirely.

**Vector:** XSS, malicious npm dependency, injected script, compromised WebView content.

**Impact:** Full access to all backend commands in scope — filesystem read/write, process execution, secret access, network requests that bypass CORS.

**Real incidents:** Tauri audit by Radically Open Security (2024) found 11 High findings on this surface. CVE-2025-31477 (CVSS 9.3/9.8, April 2025) allowed RCE via shell plugin. CVE-2024-35222 (CVSS 5.9, May 2024) allowed iframes to invoke IPC commands.

**Code review signs:**
- Bridge internals exposed as a global variable accessible to all scripts
- No per-command authentication guard
- Commands registered globally with no scope restriction
- `withGlobalTauri: true` or equivalent framework flag enabling global bridge access

---

## ATK-IPC-02 — Capability / Permission Bypass via Misconfiguration

**Description:** The framework's permission model is deny-by-default, but misconfigured capability files (wildcards, overly broad scopes, leftover test configs) silently grant excessive access.

**Vector:** Any code executing in the frontend layer — no exploit required beyond having bridge access.

**Impact:** Total bypass of the permission model; arbitrary filesystem access, command injection via sidecar arguments, SSRF via backend HTTP plugin.

**Code review signs:**
- Wildcard path scopes: `"path": "**"` grants full filesystem
- Wildcard argument scopes: `"args": true` on a subprocess allows arbitrary argument injection
- HTTP scopes allowing `http://**` enable SSRF to internal networks and cloud metadata endpoints
- Test capability files left in the production build directory
- Window scopes using `"windows": ["*"]` for sensitive permissions

---

## ATK-IPC-03 — SSRF via Backend HTTP Proxy

**Description:** The backend process proxies HTTP requests on behalf of the frontend. Requests bypass browser CORS and network restrictions because they originate from the host OS process, not the browser sandbox.

**Vector:** Frontend IPC call to a backend HTTP plugin with an attacker-controlled URL.

**Impact:** Access to cloud metadata endpoints (AWS IMDSv1: `169.254.169.254`), internal admin interfaces, private network services. NTLM hash capture via SMB redirects on Windows.

**Bypass techniques:** IPv6 loopback (`[::1]`), hex-encoded IPs (`0x7f000001`), decimal integer encoding, URL-encoded characters to bypass naive string filters.

**Code review signs:**
- HTTP capability scope includes `http://**` or any non-HTTPS scheme
- No explicit deny list for RFC 1918 ranges, loopback, and link-local addresses
- Backend HTTP calls without URL validation before dispatching

---

## ATK-IPC-04 — Path Traversal via Filesystem Scope

**Description:** Overly broad filesystem permission scopes allow reading or writing files outside the intended directory, including SSH keys, cloud credentials, and browser session data.

**Vector:** IPC call with a path argument that traverses outside the allowed base directory, or symlink that exits the allowed scope.

**Impact:** Exfiltration of `~/.ssh/id_rsa`, `~/.aws/credentials`, `~/.docker/config.json`, OAuth tokens, and platform WebView session data.

**Code review signs:**
- No symlink resolution before scope check
- Scope check done by string prefix match, not canonical path comparison
- No explicit deny for known sensitive directories (`.ssh`, `.aws`, `.gnupg`)
- Path arguments accepted from IPC without null-byte or `..` checks

---

## ATK-IPC-05 — Message Forgery (Unauthenticated IPC Channel)

**Description:** IPC between two local processes (e.g., main process and sidecar) carries no cryptographic authentication. Any process running as the same OS user can inject messages into stdin or intercept stdout.

**Vector:** Local process with same UID as the target; visible via `/proc/PID/fd/` on Linux.

**Impact:** Arbitrary command injection into the sidecar pipeline; exfiltration of secrets passed as environment variables or command-line arguments (visible in `/proc/PID/environ` and `/proc/PID/cmdline`).

**Code review signs:**
- IPC messages sent over stdin/stdout with no HMAC or signature
- Secrets passed via CLI arguments (visible to all processes with same UID)
- Secrets passed via environment variables (persist for process lifetime)
- No nonce or timestamp in IPC messages (replay possible)

---

## ATK-IPC-06 — Replay Attack on IPC Channel

**Description:** Captured valid IPC messages are replayed to re-execute privileged operations without re-authenticating.

**Vector:** Local attacker intercepts a valid signed message (e.g., "execute backup") and resends it multiple times.

**Impact:** Repeated execution of privileged operations (data deletion, payment triggers, file writes).

**Code review signs:**
- IPC signatures verified but no nonce uniqueness check
- No timestamp validation or message TTL enforcement
- Nonce cache unbounded (can be exhausted causing nonces to be evicted and reused)

---

## ATK-IPC-07 — Sidecar / Child Process Injection

**Description:** Arguments passed to a spawned child process are controlled by attacker-supplied data. Shell metacharacters or option-injection characters turn data arguments into control arguments.

**Vector:** User-controlled input flows into a subprocess invocation, especially when shell interpolation is used.

**Impact:** Arbitrary command execution with the privileges of the parent process.

**Real incident:** CVE-2024-24576 (CVSS 10.0, 2024) — `BatBadBut` — Rust `std::process::Command` on Windows failed to escape `.bat`/`.cmd` arguments, enabling full command injection. Incomplete fix tracked as CVE-2024-43402.

**Code review signs:**
- `shell=True` (Python) or equivalent in any subprocess call
- Arguments built by string concatenation rather than a list
- No `--` separator inserted before user-controlled positional arguments
- Argument validation missing or done after shell expansion

---

## ATK-IPC-08 — Deep Link Injection

**Description:** Custom URI scheme handlers (`myapp://action?param=value`) receive attacker-controlled parameters from browser links, QR codes, or other applications. Parameters are passed to backend operations without validation.

**Vector:** Malicious link opened by the user; any app on the device can register and send deep links.

**Impact:** Path traversal, SSRF, arbitrary command execution, or state manipulation depending on what the handler does with the parameters.

**Code review signs:**
- Deep link handler passes query parameters directly to filesystem or subprocess operations
- No schema validation: scheme, path, and parameter allow-list not enforced
- URL length not bounded
- `..` sequences not rejected in path components

---

## ATK-IPC-09 — Shared Memory / Socket Squatting

**Description:** IPC socket or shared memory segment created without restrictive permissions, allowing other processes to connect or race to create the socket first.

**Vector:** Process with same or different UID on the same machine.

**Impact:** Man-in-the-middle on IPC channel; privilege escalation if the legitimate server grants elevated permissions to any connecting client.

**Real incidents:** CVE-2022-21893 (Windows RDP Named Pipe, 2022) and CVE-2022-25365 (Docker Desktop, 2022) exploited default pipe permissions allowing any local user to connect.

**Code review signs:**
- Unix sockets created with default umask (world-readable)
- Windows Named Pipes created without explicit DACL (default: `Everyone` has access)
- No `FILE_FLAG_FIRST_PIPE_INSTANCE` flag (allows squatting on pipe name)
- No `SO_PEERCRED` / `LOCAL_PEERCRED` check to verify connecting process identity

---

## ATK-IPC-10 — Isolation Bypass via Encryption Key Extraction

**Description:** Sandboxed IPC isolation relies on a cryptographic key embedded in the frontend layer. If the key is marked extractable or accessible to JavaScript, an attacker with script execution can extract it and forge arbitrary messages.

**Vector:** XSS in the main application window (not the isolation iframe).

**Impact:** Complete bypass of the isolation layer; arbitrary privileged IPC calls.

**Real incident:** Radically Open Security Tauri audit (2024), finding TAU2-040: AES-GCM isolation key was marked `extractable: true` in SubtleCrypto, allowing any script in the main window to extract and reuse it.

**Code review signs:**
- Crypto key created with `extractable: true`
- Isolation layer does not apply a separate, independent trust boundary
- On some platforms (Linux, Android in certain frameworks), iframes share the same IPC key as the main window
