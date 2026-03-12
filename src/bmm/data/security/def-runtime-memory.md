# Defense Patterns: Runtime and Memory Security
> Constant-time comparison, memory zeroization, bounded collections, CSPRNG, and type-safe error handling — applicable to any language or runtime.

---

## Overview

Language-level memory safety (as in memory-safe compiled languages) eliminates buffer overflows and use-after-free, but it does not prevent:
- Timing side-channels in secret comparison
- Secret residue in RAM after garbage collection or scope exit
- OOM crashes from unbounded collections fed by external input
- Predictable "random" values from a non-cryptographic RNG
- Integer overflow producing wrong results silently
- Information leakage through error messages
- Prototype pollution in dynamic languages

These patterns apply across languages: memory-safe compiled languages (Rust, Go, Swift), garbage-collected languages (Python, JavaScript, Java, C#, Dart), and dynamically typed scripting environments.

---

## DEF-RUN-01 — Constant-Time Comparison for Secret Values

**Pattern:** Any comparison involving a secret value (token, HMAC signature, password hash, API key) uses a constant-time comparison function. Standard equality (`==`, `.equals()`, `===`) short-circuits at the first differing byte, creating a timing side-channel.

**How timing attacks work:** An attacker sends many candidate values and measures response latency. A comparison that stops at byte 0 is faster than one that reaches byte 31. Over enough samples, the attacker deduces the correct value one byte at a time.

**Language-specific implementations:**

| Language | Correct approach | Forbidden |
|---|---|---|
| Rust | `subtle::ConstantTimeEq`, `hmac::Mac::verify_slice` | `==` on `Vec<u8>`, `str` |
| Python | `hmac.compare_digest(a, b)` | `==`, `!=`, early-return loops |
| Java / Kotlin | `MessageDigest.isEqual(a, b)` | `Arrays.equals` (short-circuits) |
| Go | `subtle.ConstantTimeCompare(a, b)` | `bytes.Equal`, `==` |
| JavaScript / TypeScript | Custom XOR loop over all bytes | `===`, `Buffer.compare` |
| C# | `CryptographicOperations.FixedTimeEquals(a, b)` | `SequenceEqual`, `==` |
| Dart | Custom XOR loop; no stdlib constant-time function | `==` on `Uint8List` |

**Critical edge case — length leakage:** Returning early when lengths differ leaks the length of the expected value. Perform a dummy comparison of the same length before returning false, or always compare using a fixed-length canonical form (e.g., hash both sides with HMAC before comparing).

**Applies to:** HMAC verification, session token validation, API key comparison, password hash comparison, CSRF token comparison.

**Code review checklist:**
- [ ] No `==` or `.equals()` used to compare secrets
- [ ] Custom types implementing `PartialEq` (or equivalent) on secret values explicitly use constant-time comparison — derived equality does not
- [ ] Length mismatch does not short-circuit before performing a dummy constant-time operation

---

## DEF-RUN-02 — Memory Zeroization (Do Not Trust the Garbage Collector)

**Pattern:** Sensitive values (keys, tokens, passwords, HMAC secrets) are explicitly overwritten with zeros immediately after their last use. Do not rely on garbage collection, scope exit, or OS page reclamation to remove secret content from memory.

**Why GC is insufficient:**
- GC does not guarantee when (or if) memory is zeroed before reuse
- Core dumps, swap files, and crash reports capture live process memory — keys in memory become keys in the dump
- Memory inspection tools available to processes with the same UID can read non-zeroed pages

**Language-specific implementations:**

| Language | Correct approach |
|---|---|
| Rust | `zeroize::Zeroize` / `ZeroizeOnDrop` trait; `secrecy::SecretString` |
| Python | Overwrite `bytearray` elements with `0` in `__del__` or explicit `zeroize()` call; disable core dumps with `resource.setrlimit(RLIMIT_CORE, (0, 0))` |
| Java | Use `char[]` for passwords, call `Arrays.fill(arr, '\0')` after use; never `String` (interned, cannot be zeroed) |
| Go | `copy(slice, make([]byte, len(slice)))` immediately after use; `memguard` library for hardened buffers |
| JavaScript / TypeScript | Fill `Uint8Array` with `crypto.getRandomValues` or a zeroing loop; `String` cannot be zeroed (use `Buffer` / `Uint8Array`) |
| C# | `CryptographicOperations.ZeroMemory(span)` or `Array.Clear(arr, 0, arr.Length)` |

**Additional measures:**
- Disable core/crash dumps at process startup (OS-level: `RLIMIT_CORE = 0` on Linux, `DisableErrorReporting` on Windows)
- Avoid putting secrets into logging buffers, debug output, or serialized state
- Do not copy secrets into intermediate strings or buffers unnecessarily — minimize the number of copies in memory

**Code review checklist:**
- [ ] Secret types explicitly zeroize on `Drop`/`Finalize`/`__del__`
- [ ] No `String` (Java) used for passwords — use `char[]`
- [ ] No `console.log`, `print`, `tracing::debug!` of secret values (use redacted wrappers)
- [ ] `Debug` / `__repr__` / `toString` on secret types outputs `[REDACTED]`, not the value
- [ ] Core dumps disabled at application startup

---

## DEF-RUN-03 — Bounded Collections (MaxSize + Eviction)

**Pattern:** Any collection (map, set, list, queue, cache) whose size depends on externally controlled input has an explicit maximum size and an eviction strategy. Unbounded collections fed by external data are OOM vectors.

**Attack scenario:** An attacker sends one unique key per request (unique session IDs, unique IPs, unique nonces). The server's in-memory map grows without limit until the process crashes or is killed by the OOM killer, causing a denial of service.

**Eviction strategies by use case:**

| Use case | Recommended strategy |
|---|---|
| Nonce / replay cache | LRU with TTL: evict least-recently-seen AND expire by timestamp |
| Rate limiter per IP | Evict expired windows first; reject new callers if still full |
| Session cache | LRU + session TTL |
| DNS / URL cache | LRU with size cap |
| Event deduplication | Sliding window with time-based eviction |

**Sizing guidance:**
- Calculate the memory cost per entry: `nonce (32 bytes) + timestamp (8 bytes) + HashMap overhead (~40 bytes)` = ~80 bytes/entry. 10,000 entries = ~800 KB.
- Set the maximum based on available memory budget, not "however many fit".
- Document the maximum in code comments so it can be reviewed.

**Code review checklist:**
- [ ] Every `HashMap`, `HashSet`, `Vec`, `LruCache`, or equivalent fed by external data has a `max_size` constant
- [ ] Eviction is implemented and tested — reaching `max_size` does not cause a panic or silent data loss
- [ ] The maximum is documented with a memory-cost calculation
- [ ] Rate-limit tables use the same bounded pattern

---

## DEF-RUN-04 — CSPRNG for All Security-Sensitive Random Values

**Pattern:** Any value used for security purposes (nonce, token, session ID, CSRF token, key material, salt) is generated by a Cryptographically Secure Pseudo-Random Number Generator (CSPRNG) seeded from OS entropy.

**Why standard RNGs are insufficient:** General-purpose RNGs (e.g., `Math.random()`, `rand::thread_rng()`, `java.util.Random`, `random.random()`) are designed for speed and statistical distribution, not unpredictability. An attacker who observes enough output can reconstruct the internal state and predict future values.

**Language-specific CSPRNG:**

| Language | Correct | Forbidden for secrets |
|---|---|---|
| Rust | `OsRng.fill_bytes()`, `ChaCha20Rng::from_rng(OsRng)` | `rand::thread_rng()`, `rand::random()` |
| Python | `os.urandom(n)`, `secrets.token_bytes(n)` | `random.random()`, `random.randint()` |
| Java | `SecureRandom.getInstanceStrong()` | `java.util.Random`, `Math.random()` |
| Go | `crypto/rand.Read(buf)` | `math/rand` |
| JavaScript | `crypto.getRandomValues(buf)`, `crypto.randomBytes(n)` (Node.js) | `Math.random()` |
| C# | `RandomNumberGenerator.GetBytes(buf)` | `System.Random` |
| Dart | `dart:math` `Random.secure()` | `Random()` without `.secure()` |

**Applies to:** Session tokens, CSRF tokens, nonces, HMAC secrets, API keys generated server-side, password reset tokens, OAuth state parameters.

**Code review checklist:**
- [ ] No `Math.random()`, `random.random()`, or non-secure RNG used for any security value
- [ ] Token generation uses the OS entropy source directly or a CSPRNG seeded from it
- [ ] Random seeds are never hardcoded (prevents reproducibility attacks)

---

## DEF-RUN-05 — Integer Overflow Protection

**Pattern:** Arithmetic on values derived from external input is checked for overflow before use. Use saturating or checked arithmetic; avoid unsigned subtraction that can wrap to a large number (bypasses TTL checks). In JavaScript, use `BigInt` for integers above 2^53. In Java/C#/Go use `Math.addExact()` or explicit pre-checks — both silently wrap on overflow.

**Code review checklist:**
- [ ] Offset and size calculations on external data use checked/saturating arithmetic
- [ ] No unsigned subtraction where the result could wrap
- [ ] JavaScript code using large integers uses `BigInt` where precision matters

---

## DEF-RUN-06 — Prototype Pollution Prevention (Dynamic Languages)

**Pattern:** In prototype-based languages (JavaScript, etc.), user-supplied JSON containing `__proto__`, `constructor`, or `prototype` as keys silently modifies the prototype of all objects in the process, potentially bypassing security checks. `{ "__proto__": { "isAdmin": true } }` merged into any object makes every object have `isAdmin === true`.

**Defenses:** Validate and reject forbidden keys before any merge; use `Object.freeze()` on configuration objects after initialization; use `Object.create(null)` for lookup tables; prefer library merge functions that include prototype-pollution fixes over custom recursive merge.

**Code review checklist:**
- [ ] JSON from external sources is validated against a schema before use
- [ ] No custom `deepMerge` without `__proto__` key rejection
- [ ] Configuration objects are frozen after initialization

---

## DEF-RUN-07 — Type-Safe Error Handling (No Empty Catch Blocks)

**Pattern:** Every error is caught, logged internally with full detail, and then either handled or converted to a safe generic response. Empty `catch` blocks that discard errors are forbidden. Security operations fail CLOSED: an error blocks the operation, never silently passes it.

**Fail-closed principle:** If a security check (signature verification, permission check, input validation) throws an error, the operation is blocked — the error is not treated as "check passed".

**Safe error mapping:** Internal errors logged in full detail internally; external responses carry only a generic error code from a closed enumeration — no paths, table names, library versions, or stack traces.

**Bad patterns:** `catch (_) {}` (silent swallow), `catch (e) { return true; }` (fail-open), `catch (e) { throw Error(e.stack) }` (leak), `unwrap()` in production paths (panic = DoS).

**Code review checklist:**
- [ ] No empty `catch` blocks in any security-relevant code path
- [ ] Errors from security checks block the operation — never fall through
- [ ] Internal error types are never serialized directly to external callers
- [ ] User-visible error messages are static strings — no interpolation of internal values

---

## DEF-RUN-08 — Safe String Handling

**Pattern:** String operations on externally supplied values validate length, encoding, and character set before use. Null bytes, path separators, and control characters are rejected at the validation boundary.

**Common failure points:**
- Null byte injection: `filename\0.txt` — the OS sees `filename`, the application sees `filename\0.txt`
- Path traversal: `../../etc/passwd` embedded in a filename field
- Encoding confusion: over-long UTF-8 sequences that decode to ASCII control characters
- Windows device names: `CON`, `NUL`, `COM1` as filenames cause OS-level issues

**Validation order:** null bytes, then maximum length, then path separators and `..`, then encoding well-formedness, then character-set allowlist regex. For file paths: resolve to canonical form and verify within the allowed base directory.

**Code review checklist:**
- [ ] Null byte check is the first validation step on any string field
- [ ] Path fields use canonical resolution to detect traversal, not string prefix matching
- [ ] Windows reserved names (`CON`, `NUL`, `COM1`–`COM9`, `LPT1`–`LPT9`) are rejected for file paths
- [ ] String length is bounded before regex matching (long strings can cause ReDoS on poorly written regex)

