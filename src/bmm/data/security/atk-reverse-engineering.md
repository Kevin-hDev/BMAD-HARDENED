# Attack Patterns: Reverse Engineering

> How attackers extract, analyze, and modify compiled applications and binaries.

---

## Overview

Reverse engineering (RE) attacks target the compiled artifacts of an application rather than
its running instance. The attacker's goal varies: extract hardcoded secrets, understand internal
logic, identify exploitable code paths, or create a trojanized replacement binary.

**Attacker motivations ranked by frequency:**
1. Extract credentials/keys hardcoded or recoverable from binaries.
2. Understand authentication and license validation logic to bypass it.
3. Identify attack surface (IPC commands, API endpoints, data formats).
4. Create a backdoored replacement binary for distribution.
5. Recover proprietary algorithms or business logic.

**Key insight:** The difficulty of RE varies dramatically by technology. Python bytecode
(PyInstaller) is near-trivially recoverable. Stripped, optimized native binaries (Rust,
C++) require expertise but yield to determined attackers.

---

## Binary Extraction and Decompression

### Bundled Python (PyInstaller, cx_Freeze, Nuitka)

PyInstaller appends a `CArchive` to a native bootloader. Tools locate the magic cookie
automatically and extract all bytecode files with corrected headers.

**Tool:** pyinstxtractor v2.0 / pyinstxtractor-ng — extracts `.pyc` from any PyInstaller 2.0-6.16.0
binary in under 60 seconds. PyLingual decompiles 77-87% perfectly (Python 3.6-3.13). Variable
names always preserved in bytecode. The `--key` AES option (removed v6.0.0) was never effective —
key stored inside the same binary. CVE-2025-59042 (CVSS 7.0): code injection via `sys.path` in
`_MEIPASS`; requires PyInstaller >= 6.10.0.

### Electron / ASAR and Native Binaries

- **Electron ASAR:** `npx asar extract` — no crypto protection; source maps expose full TS source.
- **Native (C/C++/Rust/Go):** Harder but analyzable. Key factors: debug symbols present? panic
  strings present? LTO applied? IDA Pro, Ghidra, Binary Ninja with language-specific plugins.

---

## Debug Attachment

Attacker attaches a debugger to inspect runtime state, extract keys from memory,
or modify execution flow.

### Debugger Attachment by Platform

**Linux:** `ptrace(PTRACE_ATTACH, pid)` requires same UID or root. Reading `/proc/PID/mem`
requires the same UID. No privilege escalation needed for user-level processes.

**Windows:** `OpenProcess(PROCESS_VM_READ, ...)` plus `ReadProcessMemory` requires same
integrity level. Tools: x64dbg, WinDbg.

**macOS:** `taskport` access requires same UID or root. LLDB is the primary tool.

### Debugger Detection Bypass

Applications that detect debuggers can be bypassed:

| Detection method | Bypass |
|-----------------|--------|
| `ptrace(PTRACE_TRACEME)` — fails if already traced | LD_PRELOAD hook returning success; patch the syscall call to NOP |
| `/proc/self/status TracerPid != 0` | LD_PRELOAD hook on `read()` returning falsified content |
| `IsDebuggerPresent()` (Windows) | Patch PEB.BeingDebugged byte to 0; ScyllaHide plugin (x64dbg) patches 20+ checks automatically |
| `sysctl KERN_PROC P_TRACED` (macOS) | DYLD_INSERT_LIBRARIES hooking `sysctl`; NOP the `ptrace(PT_DENY_ATTACH)` call |
| Timing checks (rdtsc, QueryPerformanceCounter) | ScyllaHide normalizes timing counters |

**Tool:** ScyllaHide (x64dbg plugin) — single activation bypasses all standard Windows
anti-debug techniques simultaneously.

---

## Memory Dumping

| Platform | Method | Privilege needed |
|----------|--------|-----------------|
| Linux | `/proc/PID/mem` at heap offsets (from `/proc/PID/maps`) | Same user |
| Windows | `procdump.exe -ma <PID>` | Same integrity level |
| macOS | `lldb -p <PID>` + `memory read` | Blocked by Hardened Runtime |

**What attackers extract:** API key patterns (`sk-`, `gsk_`), 32/64-byte high-entropy keys,
connection strings with credentials. Secrets not zeroed after use are always recoverable.
GC languages (Python, Java, Go) do not guarantee timely zeroing.

---

## API Hooking and Interception

| Mechanism | Platform | How |
|-----------|----------|-----|
| LD_PRELOAD | Linux | Intercept any libc/dynamic function; no privilege for non-SUID binaries; persistence via `/etc/ld.so.preload` (root) |
| DYLD_INSERT_LIBRARIES | macOS | Same as LD_PRELOAD; blocked by Hardened Runtime unless `allow-dyld-environment-variables` entitlement set |
| DLL Hijacking | Windows | Place malicious DLL in earlier search-order path than legitimate DLL; discover with ProcMon `NAME NOT FOUND` filter |
| Frida | Cross-platform | Inject JS engine into any process; hook any function by name or address at runtime without restarting |

**Common targets:** `open()`, `read()`, `connect()`, crypto/KDF functions, `sqlite3_exec()`.

### Frida (Cross-platform dynamic instrumentation)

Frida injects a JavaScript engine into any process and allows hooking any function
by name or address, at runtime, without modifying the binary.

```
frida -p <PID> -l hook.js
```

Frida is cross-platform (Linux, Windows, macOS, iOS, Android) and does not require
the application to be restarted.

---

## Obfuscation Bypass

### JavaScript / Frontend

Standard minification (Terser, esbuild) is not obfuscation — it reduces size, not
readability. String constants, API endpoints, error messages, and route names remain
readable in any minified bundle.

True obfuscation tools (obfuscator.io) are defeated by automated deobfuscators
(deobfuscate.io, JS Nice, AST-based analysis).

**Source maps** (`*.js.map`): If included in production builds, they restore the original
TypeScript/JavaScript source verbatim. Attacker command: download the `.js.map` file
referenced in the bundle comment, then use `source-map-extractor`.

### Native Binaries — Information Leakage

Even without symbols, native binaries leak information:

| Leak source | Information revealed |
|-------------|---------------------|
| Panic strings / `assert` messages | Source file paths (`src/auth/login.rs:42`), module structure |
| String constants in `.rodata` | API endpoints, error messages, format strings |
| Import table | All external library functions called |
| Exception handler tables | Control flow structure |
| Debug information (DWARF) | Full source mapping if `debug = true` in build config |

**Panic strings in Rust:** `core::panic::Location` embeds source file paths and line numbers.
Recoverable via `strings ./binary | grep -E 'src/.*\.rs:[0-9]+'`. Script-extractable
automatically (documented by Binary Ninja community, 2024).

### Symbol Stripping Bypass

Stripped binaries are analyzed via signature matching (IDA Pro FLIRT, Ghidra function ID,
IDARustler 2025 for Rust). Stripping helps but does not prevent RE — behavior analysis
does not require function names.

---

## Code Signing Bypass

| Method | How it works |
|--------|-------------|
| Self-signed cert | Match publisher Common Name; fools apps that check presence, not CA chain |
| Stolen EV cert | EV certs sold on dark web markets; signed binaries pass SmartScreen |
| Hardcoded public key patch | Extract public key from updater binary, replace with attacker's key, sign malicious update |

Condition for key patch: no external integrity check on the binary before patching.

---

## Canary and Honeypot Detection by Attackers

| Technique | How |
|-----------|-----|
| Temporal correlation | Access files one at a time with delays; observe which triggers an alert |
| Source code analysis | Search public repo: `grep -r "canary\|honeypot\|decoy"` — reveals all canary names |
| Content validity test | Try each found credential; all fail = likely canary |
| Timestamp analysis | Canary files share creation timestamps; real credentials have varied timestamps |
| Indirect listing | `stat`/`find` do not trigger `inotify IN_OPEN`; enumerate without triggering read alerts |

**Key mitigation:** Generate canary file paths at deployment from a per-installation random
seed; never store canary names in source code.

---

## Attacker RE Workflow

1. **Discovery** — `file`, `strings`, check for debug info, source maps, panic strings.
2. **Extraction** — pyinstxtractor (Python), `npx asar extract` (Electron), IDA/Ghidra (native).
3. **Analysis** — Map auth logic, IPC commands, secret handling, exploitable call patterns.
4. **Runtime** — LD_PRELOAD / Frida hooks; breakpoint on crypto calls to capture keys in memory.
5. **Action** — Use recovered secrets directly, or replace binary with trojanized version.
