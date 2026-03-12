# Attack Patterns: Privilege Escalation

> Universal privilege escalation techniques across Linux, Windows, and macOS — real CVEs, attack chains, and indicators of vulnerable code patterns.

---

## 1. Linux — polkit Chain CVE-2025-6018 / 6019 / 6020

**Discovered by:** Qualys, June 2025
**CVSS:** CVE-2025-6018: 7.8, CVE-2025-6019: 7.8, CVE-2025-6020: TBD
**Status:** Patched June 2025
**Prerequisites:** Valid SSH access (non-root), udisks2 installed (present in ~98% of enterprise distributions)

**Attack chain:**

1. Write to `~/.pam_environment`:
   ```
   XDG_SEAT=seat0
   XDG_VTNR=1
   ```
2. Linux-PAM (since v1.3.0) reads this file and exports the variables into the SSH session environment (CVE-2025-6018).
3. polkit evaluates the `allow_active` policy by checking `XDG_SEAT` and `XDG_VTNR` — the attacker is now treated as a physically-present console user.
4. The `allow_active` trust zone unlocks actions normally denied to remote SSH users.
5. CVE-2025-6019: libblockdev omits the `nosuid` flag on temporary mounts via udisks2 — permits mounting a filesystem with the SUID bit active.
6. Execute a SUID binary from that mount — gains root.

**Impact:** Full root without touching `pkexec`. Completely bypasses the standard polkit model. Any application that calls `pkexec` and relies on `allow_active` is affected.

**CVE-2025-6020 (bonus):** Path traversal in `pam_namespace` via symlinks and race conditions — an additional escalation path in hardened environments.

**What to look for in code review:**
- polkit policy files using `allow_active` anywhere — should use `auth_admin` exclusively
- `user_readenv` enabled in PAM configuration (must be disabled to block the `~/.pam_environment` injection)
- Applications that invoke pkexec without first clearing the environment

---

## 2. Linux — PwnKit CVE-2021-4034 (Still Unpatched on Many Systems)

**CVSS:** 7.8
**Status:** Patched 2022 — still unpatched on many legacy enterprise systems
**Vulnerability present since:** 2009 (pkexec initial release)

**Mechanism:** Out-of-bounds write in `pkexec`'s handling of `argv`. The vulnerability allows overwriting environment variables to inject `LD_PRELOAD` pointing to a malicious shared library. pkexec loads the library with root privileges.

**Attack steps:**
1. Create a directory whose name becomes `argv[0]` during pkexec execution.
2. Exploit the OOB write to overwrite environment variable memory.
3. Inject `LD_PRELOAD` pointing to a malicious `.so` file.
4. pkexec loads the library as root — shell obtained.

**Impact:** Root from any local or SSH session. Public exploit PoC available from Qualys since 2022.

**What to look for in code review:**
- Any system where pkexec is available and unpatched (check package version against distribution advisories)
- Applications that invoke pkexec with user-controlled arguments
- Environment variables not cleared before pkexec invocation (`LD_PRELOAD`, `PATH` injection)

---

## 3. Windows — UAC Bypass (80+ Active Methods)

**Context:** Microsoft officially considers UAC a non-security boundary. Bypass techniques do not receive security patches. As of Windows 11 24H2 (2025), 80+ active bypass methods are catalogued in the UACME repository (hfiref0x/UACME, 7400+ stars).

### 3a. CMSTPLUA COM Auto-Elevation (CLSID: 3E5FC7F9-...)

**Used by:** Formbook, LockBit, Cobalt Strike

**Steps:**
1. Instantiate the CMSTPLUA COM object via `CoCreateInstance`.
2. This object is marked `auto-elevate` in its manifest — Windows elevates it without showing a UAC prompt.
3. Use the `ICMLuaUtil` interface to execute arbitrary commands with elevated privileges.

**Impact:** SYSTEM-level execution with no visible UAC prompt.

### 3b. Fodhelper Registry Hijacking

**Used by:** BlankGrabber, Glupteba

**Steps:**
1. Write to `HKCU\Software\Classes\ms-settings\shell\open\command` (writable by the current user).
2. Set `DelegateExecute` to an empty string in the same key.
3. Launch `fodhelper.exe` (marked `requireAdministrator` + `autoElevate=true`).
4. fodhelper reads the registry key, auto-elevates, and executes the payload.

### 3c. SilentCleanup / DismHost DLL Hijacking (UACME Method 34)

**Confirmed working:** Windows 11 24H2, February 2025 (after Microsoft's pseudo-fix)

**Steps:**
1. The `SilentCleanup` scheduled task runs at `highestAvailable` privilege without a UAC prompt.
2. `DismHost.exe` (launched by SilentCleanup) searches for DLLs in user-writable paths.
3. Place a malicious DLL in the search path.
4. DismHost loads the DLL with elevated privileges.

### 3d. UAC Fatigue (Psychological Attack)

**Used by:** DCRat, PureMiner

**Technique:** Trigger a loop of UAC prompts at the moment when users expect legitimate application prompts. Users click "Yes" out of conditioning.

**Critical risk for applications that show regular elevation prompts:** A malicious process can time its fake prompt to coincide with the application's legitimate prompt, creating dialog confusion.

---

## 4. Windows — Disabling Defender / BYOVD

**Defendnot (May 2025):** Abuses the undocumented Windows Security Center API. Inject a DLL into `Taskmgr.exe` (a Microsoft-signed PPL binary), call the WSC API from that trusted context to register a fake AV product — Windows automatically disables Defender to avoid conflicts. Zero user-visible indication. Related: CVE-2024-20671.

**BYOVD (Bring Your Own Vulnerable Driver) — Active since mid-2024:** Load a legitimately signed but vulnerable Windows driver (e.g., TrueSight anti-rootkit v2.0.2 — one of 2500+ variants on LOLDrivers / loldrivers.io). The signed driver is accepted even with Secure Boot active. Exploit the driver vulnerability to execute code in kernel mode, then terminate EDR processes, disable their services, and modify their kernel callbacks. Used by EDRKillShifter (RansomHub, August 2024), later adopted by Medusa, BianLian, Play.

**CrowdStrike incident (July 19, 2024):** Not an attack, but illustrates systemic risk: Channel File 291 defect (uninitialized pointer) caused BSoD on ~8.5 million machines. $10+ billion in estimated losses. Security tooling is itself a high-impact attack surface.

---

## 6. macOS — TCC Bypass CVE-2025-43530 and CVE-2025-31250

### 6a. CVE-2025-43530 — ScreenReader.framework MIG Service Bypass

**Service:** MIG `com.apple.scrod` in ScreenReader.framework
**Core defect:** Identity validation uses `SecStaticCodeCreateWithPath` (validates a file path on disk) instead of audit tokens (bound to the running process). An attacker can supply the path of a legitimately signed Apple binary while being a different process.

**Steps:**
1. Identify a signed Apple binary on the system (e.g., `/usr/bin/osascript`).
2. Send a MIG message to `com.apple.scrod` with this path as proof of identity.
3. The service validates the path (not the actual process) and grants permissions.
4. Result: arbitrary AppleScript execution, AppleEvents to any process, access to microphone, camera, and documents — without any user interaction.

**Impact:** Full TCC bypass with silent access to all protected data.

### 6b. CVE-2025-31250 — TCC Consent Dialog Spoofing

**Status at discovery:** Unpatched on macOS Ventura 13.7.6 and Sonoma 14.7.6
**Mechanism:** Any application can send a forged XPC message to `tccd` (the TCC daemon) that displays a prompt appearing to come from legitimate application A but actually grants permissions to malicious application C.

**Impact:** The user believes they are authorizing a known application. They are in fact authorizing the attacker.

---

## 7. macOS — Sandbox Escape CVE-2025-31191

**Discovered by:** Microsoft Threat Intelligence, March 2025
**Mechanism:** Exploitation of security-scoped bookmarks to escape the macOS application sandbox.

**Steps:**
1. A sandboxed application creates a security-scoped bookmark for a file it has legitimate access to.
2. A weakness in bookmark resolution is exploited to access paths outside the sandbox boundary.
3. The application accesses filesystem resources normally inaccessible from a sandboxed context.

**Impact:** Full sandbox escape. Access to the unrestricted filesystem.

---

## 8. Pattern — Unauthenticated IPC to Privileged Helper (CCleaner)

**Discovered by:** Quarkslab, March 2025. CCleaner's Privileged Helper Tool exposed a Unix socket at `/var/run/com.piriform.ccleaner.CCleanerAgent.socket` with `0666` permissions and no caller authentication. Any local process could connect and issue commands executed as root.

**Universal failure mode:** World-writable IPC socket or named pipe + no caller authentication + arbitrary string commands = any local process owns root. The "Project LOST" catalogue (0xanalyst.github.io/Project-Lost/) documents systematic abuse of security tools as LPE vectors.

**What to look for:** IPC socket permissions `0666`, absence of HMAC or code signing verification on incoming commands, helper that accepts free-form strings rather than a closed action enum.

---

## 9. Pattern — Rollback to Weakened State and Environment Injection

**Rollback attacks (any app storing pre-hardening baseline):**
- **Scenario A:** Modify the state snapshot before hardening runs → rollback restores a weaker-than-original state
- **Scenario B:** Corrupt the state file → system is trapped in an intermediate, unstable state
- **Scenario C (replay):** Save state at T1 (weak), replace state file after hardening at T2 → system reverts to T1

**What to look for:** State files without cryptographic signatures, no hash chaining between snapshots, rollback logic that does not check whether the target state is less secure than the current state, state files with user-writable permissions.

**Environment injection (helper binary exploitation):** When a privileged helper is launched via `pkexec`, `sudo`, or OS elevation APIs, it inherits the calling process's environment. `LD_PRELOAD`, `DYLD_INSERT_LIBRARIES`, `PATH`, `PYTHONPATH`, and `XDG_*` variables can all redirect execution to attacker-controlled code (as exploited in CVE-2025-6018 via `HOME`/`XDG_SEAT`).

**What to look for:** Process spawns that do not call `env_clear()` before elevation, helpers that resolve command names via `PATH` instead of absolute paths.

---

## CVE Reference Table

| CVE | OS | Type | CVSS | Status | Summary |
|---|---|---|---|---|---|
| CVE-2025-6018 | Linux | LPE via polkit | 7.8 | Patched June 2025 | PAM `~/.pam_environment` injection → `allow_active` bypass |
| CVE-2025-6019 | Linux | LPE via udisks2 | 7.8 | Patched June 2025 | libblockdev omits `nosuid` → SUID mount exploitation |
| CVE-2025-6020 | Linux | LPE via pam_namespace | TBD | Patched June 2025 | Path traversal via symlinks and race conditions |
| CVE-2021-4034 | Linux | LPE via pkexec | 7.8 | Patched 2022; unpatched on legacy | PwnKit — OOB write → LD_PRELOAD injection |
| CVE-2025-43530 | macOS | TCC bypass | Critical | Patched | ScreenReader.framework — path vs. audit token validation |
| CVE-2025-31250 | macOS | TCC dialog spoof | Critical | Unpatched on Ventura 13.7.6 / Sonoma 14.7.6 | Forged XPC → misleading TCC consent prompt |
| CVE-2025-31191 | macOS | Sandbox escape | High | Patched | Security-scoped bookmark exploitation |
| CVE-2024-20671 | Windows | Defender bypass | High | Patched | Defender protection feature bypass |
| BYOVD TrueSight | Windows | Kernel-level EDR kill | — | Ongoing | Signed vulnerable driver → EDR process termination |
