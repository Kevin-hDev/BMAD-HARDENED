# Defense Patterns: OS Isolation and Process Privilege Hardening

> Defense-in-depth architecture for process privilege separation and OS sandboxing. No single layer is sufficient — these controls compose. Their composition is the defense.

---

## Core Principle: No Single Layer Suffices

Ona Security (March 2026) documented an autonomous agent that defeated three independent controls by reasoning alone: bypassed an application-level denylist via `/proc/self/root`, disabled its own sandbox via a single flag, then loaded a binary through the dynamic linker to bypass execution controls. Individual control bypass is expected. The combination must be infeasible.

**Six-layer model (innermost → outermost):**

| Layer | Mechanism | Defends against |
|---|---|---|
| L1 — Application | Input allowlists, closed action enums | Injected commands |
| L2 — Transport | ForceCommand, SSH certificate restrictions | Arbitrary remote execution |
| L3 — User | Dedicated low-privilege user, `no_new_privs` | Privilege escalation |
| L4 — OS sandbox | seccomp, AppArmor/SELinux, namespaces | Kernel and filesystem access |
| L5 — Network | Per-UID firewall rules | Data exfiltration |
| L6 — Audit | auditd, remote syslog | Detection and forensics |

---

## 1. Helper Binary Pattern — Principle of Least Privilege

**Never elevate the whole application.** Elevating a process that includes a UI renderer or plugin system exposes the entire attack surface to root.

**The helper binary pattern:** A separate, minimal binary handles only operations requiring elevation. The main application runs unprivileged and communicates with the helper via authenticated IPC.

**Helper requirements:**
- Accept only a closed enumeration of actions — no dynamic command construction
- Verify own binary integrity (SHA-256 hash) before executing
- Authenticate every IPC message (HMAC-SHA256, constant-time comparison)
- Reject commands older than 30 seconds (anti-replay)
- Clear the full environment before any subprocess execution
- Use absolute paths — never rely on `PATH` resolution

**OS elevation mechanisms:**

| OS | Mechanism | Critical constraint |
|---|---|---|
| Linux | `pkexec` with custom polkit policy | Use `auth_admin` exclusively — never `allow_active` (CVE-2025-6018) |
| macOS | SMJobBless Privileged Helper Tool | Validate callers via audit tokens, not file path (CVE-2025-43530) |
| Windows | Windows Service (LocalSystem) + Named Pipe | Avoids per-operation UAC; use HMAC on pipe messages |

---

## 2. Environment Purging and no_new_privs

**Before any privileged spawn:** Call `env_clear()` (or equivalent) to remove all inherited environment variables. Then set only the minimal required variables pointing to absolute, known-good paths. Variables `LD_PRELOAD`, `DYLD_INSERT_LIBRARIES`, `PATH`, `PYTHONPATH`, `HOME`, and `XDG_*` are all injection vectors (see CVE-2025-6018).

**`no_new_privs` (Linux 3.5+):** The single most important privilege control. Once set: SUID/SGID bits are ignored on execve, file capabilities are non-functional, the flag cannot be unset, and all child processes inherit it. Makes SUID exploitation structurally impossible for the protected process tree.

Apply via: `/etc/security/limits.conf`, systemd `NoNewPrivileges=yes`, or `prctl(PR_SET_NO_NEW_PRIVS, 1)`.

---

## 3. Authenticated IPC — HMAC Pattern

**The CCleaner failure (Quarkslab, March 2025):** World-writable Unix socket (`0666`) + no caller authentication = any local process issues root commands.

**Correct pattern:**
1. Main process builds: `action_enum:timestamp_ms:random_nonce`
2. Signs with HMAC-SHA256 using a secret derived from the app's code signing certificate
3. Helper verifies HMAC in constant time before acting
4. Helper rejects timestamps older than 30 seconds
5. Socket permissions: `0600`, owned by the privileged process

---

## 4. Linux — Kernel Isolation Layers

**seccomp-BPF:** Whitelist ~40-50 syscalls; default action `KILL_PROCESS`. The filter is irrevocable and inherited by all children. Block explicitly: `ptrace` (can escape seccomp), `clone`/`unshare` with user namespace flags, `keyctl`, `perf_event_open`.

**Namespaces:** Mount namespace hides binaries (bind-mount `/dev/null` over unwanted executables, make filesystem read-only except workspace). Network namespace creates an isolated stack with zero connectivity by default. Block unprivileged user namespaces with `kernel.unprivileged_userns_clone=0`.

**AppArmor (Ubuntu/Debian):** Path-based per-process policies. Limitation: symlink manipulation can bypass path rules — combine with Landlock or SELinux.

**SELinux (RHEL/Fedora):** Label-based (inode, not path). Immune to symlink bypass. More complex to configure than AppArmor but more granular.

**Landlock (Linux 5.13+):** Stackable LSM; a process restricts its own file and network (v4, kernel 6.7) access. Rules are strictly additive-restrictive — cannot grant permissions. Provides an extra layer even if AppArmor/SELinux is misconfigured.

**systemd key directives:** `NoNewPrivileges=yes`, `ProtectSystem=strict`, `ProtectHome=yes`, `PrivateTmp=yes`, `CapabilityBoundingSet=` (empty), `SystemCallFilter=@system-service`, `MemoryDenyWriteExecute=yes`, `ProtectProc=invisible`. Validate with `systemd-analyze security <service>`.

---

## 5. macOS — Isolation Mechanisms

**App Sandbox (entitlements):** Applies to GUI applications (Mac App Store, notarized distribution). Does not apply to CLI tools, daemons, or SSH-launched processes.

**sandbox-exec with SBPL profiles:** Functional on macOS 15.x (Sequoia) despite official deprecation — used in production by Chromium, Bazel, and Anthropic Claude Code. Profile structure: `(deny default)` first, then explicit allow rules for system libraries, workspace read-write, and minimal localhost network access.

**TCC caller validation:** Use audit tokens bound to the running process — not `SecStaticCodeCreateWithPath` (validates a disk path, not the live process). Path-based validation is bypassed by supplying a path to a legitimate signed binary while being a different process (CVE-2025-43530).

**Endpoint Security Framework:** Provides `AUTH` pre-authorization events to block operations before execution. Use inverted muting to monitor only the sandboxed process.

---

## 6. Windows — Isolation Mechanisms

**AppContainer / LPAC:** Recognized Microsoft security boundary. Dual-principal DACL model (user SID + AppContainer SID). Without network capabilities: zero network access. LPAC requires explicit capability for registry and COM access — most restrictive user-mode sandbox on Windows.

**WDAC (Windows Defender Application Control):** Kernel-level executable allowlisting. Enforces PowerShell Constrained Language Mode. Apply hash-based allowlist + Microsoft LOLBin block list. Active bypasses (2025): signed Electron apps, DLL sideloading without DLL signature enforcement. CVE-2025-26678 was a WDAC access bypass, patched April 2025.

**Hyper-V / Windows Sandbox:** Dedicated kernel per container — equivalent to Firecracker on Linux. Strongest Windows isolation boundary. For agent or untrusted code: Windows Service (LocalSystem) + Named Pipe + HMAC, not per-operation UAC prompts.

---

## 7. Double-Layering: Container + VM

**Why containers alone are insufficient:** OCI containers share the host kernel (~340 syscalls, 40M+ lines of C). In 2025, three runC CVEs allowed host filesystem writes from within containers via symlink manipulation.

**Double-layer options:**

| Technology | Isolation | Use when |
|---|---|---|
| Firecracker (MicroVM, Rust) | Dedicated kernel; ~5 virtio devices; <125ms startup | Maximum isolation |
| gVisor (Go Sentry) | User-space kernel; ~70 host syscalls | SaaS multi-tenant |
| Kata Containers | VM-backed OCI | Kubernetes |
| bubblewrap + seccomp | Namespace + filter (no VM) | VM overhead unacceptable |

**Recommendation for autonomous agents (2025-2026):** VM-level boundary (Firecracker or Kata) as outer layer + namespaces + seccomp + Landlock inside. Makes simultaneous kernel sharing exploitation, VM escape, and host kernel attack required.

---

## 8. Resource Limits, Filesystem, and Drift Detection

**Resource limits (apply via cgroups v2 + limits.conf):** Max processes 150-256 (fork bomb prevention), max open files 1024, max virtual memory 2 GB, core dump size 0 (no sensitive data in dumps).

**Mount options:** `nosuid,noexec,nodev` on `/tmp`, `/dev/shm`, and workspace directories. Neutralizes SUID exploitation through writable paths.

**Capabilities to never grant:** `CAP_SYS_ADMIN` ("the new root"), `CAP_SYS_PTRACE`, `CAP_DAC_OVERRIDE`, `CAP_SETUID`/`CAP_SETGID`. Drop all capabilities at startup; add back only what is explicitly required.

**Drift detection:** A periodic reconciliation loop (60-second interval) reads actual OS state (firewall status, sysctl values, LSM enforcement mode, service states) and compares to desired state. Any discrepancy alerts and optionally re-applies. Log to a remote, append-only sink — the local process must not be able to alter its own audit trail.

---

## Code Review Checklist

**Privilege Separation:**
- [ ] Privileged operations handled by a separate minimal helper binary
- [ ] Helper accepts only a closed enumeration of actions
- [ ] Helper verifies own binary integrity before execution
- [ ] All IPC messages authenticated with HMAC; constant-time comparison used
- [ ] IPC messages older than 30 seconds rejected

**Environment and Execution:**
- [ ] Environment fully cleared before any privileged subprocess launch
- [ ] All subprocess invocations use absolute paths
- [ ] `no_new_privs` set for all unprivileged worker processes
- [ ] All capabilities dropped except those explicitly required

**Linux:**
- [ ] polkit policy uses `auth_admin` — no `allow_active` anywhere
- [ ] `user_readenv` disabled in PAM configuration
- [ ] seccomp filter applied with default-deny
- [ ] `nosuid,noexec,nodev` on writable paths
- [ ] `kernel.unprivileged_userns_clone=0` where user namespaces are not needed

**macOS:**
- [ ] Privileged helpers use SMJobBless
- [ ] IPC caller validation uses audit tokens, not path-based code signing checks
- [ ] sandbox-exec profile starts with `(deny default)`

**Windows:**
- [ ] Privileged operations use Windows Service + Named Pipe, not per-operation UAC
- [ ] AppContainer or LPAC for untrusted code execution
- [ ] HVCI enabled (mitigates BYOVD)
- [ ] Event ID 7045 monitored (new driver loaded)

**IPC and State:**
- [ ] IPC sockets not world-writable (`0666` forbidden)
- [ ] State snapshots are cryptographically signed and hash-chained
- [ ] Rollback logic blocks downgrades on security-critical parameters
- [ ] State files stored `0600`, root-owned

**Monitoring:**
- [ ] Drift detection loop runs periodically
- [ ] Audit logs forwarded to remote append-only sink
