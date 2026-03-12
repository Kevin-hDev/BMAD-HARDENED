# Defense Patterns: Network and TLS Hardening

> Universal defensive controls for TLS configuration, redirect policies, DNS protection, timeouts, and API communication security — applicable to any application stack.

---

## 1. TLS Version Enforcement

**Principle:** Enforce a minimum TLS version at the application layer, independent of OS or library defaults. Do not rely on default configurations.

**Recommended floor:** TLS 1.3 minimum for all new connections. TLS 1.2 is acceptable only when TLS 1.3 is genuinely unavailable at the peer, with explicit justification.

**Why TLS 1.3:** Eliminates all CBC cipher suites (BEAST, Lucky13), removes RSA key exchange (POODLE variants), enforces Perfect Forward Secrecy by design, and reduces handshake round trips.

**What to configure:**
- Set the minimum protocol version explicitly — do not inherit from OS defaults
- Disable TLS 1.0 and 1.1 unconditionally
- Prefer a pure-Rust or pure-Go TLS backend over system OpenSSL where available to eliminate C memory safety vulnerabilities (see RUSTSEC-2025-0004: use-after-free in Rust's `openssl` crate / `native-tls` backend, fixed by migrating to `rustls`)

**Code review checklist:**
- [ ] Minimum TLS version is set explicitly in code, not assumed from defaults
- [ ] No fallback path exists that allows TLS 1.0 or 1.1
- [ ] TLS backend library is pinned to a specific version and audited

---

## 2. Certificate Validation — Fail Closed

**Principle:** TLS errors must terminate the connection. There must be no fallback to HTTP, no "accept anyway" path, and no hostname verification bypass.

**Patterns to prohibit absolutely:**
- Disabling certificate verification globally or per-host
- Disabling hostname verification
- Falling back to HTTP when HTTPS fails
- Accepting self-signed certificates in production code paths

**Fail-closed design:** Any connection failure is an error that propagates up. The caller decides whether to retry — the HTTP client never silently degrades.

**Automated verification:** Use third-party test endpoints (e.g., `expired.badssl.com`, `self-signed.badssl.com`, `wrong.host.badssl.com`) in integration tests to assert that the client correctly rejects invalid certificates.

**Code review checklist:**
- [ ] Search for any flag that accepts invalid certificates — must not appear in production code
- [ ] Search for any flag that disables hostname verification
- [ ] Verify no HTTP fallback exists in any code path
- [ ] Confirm TLS errors propagate and are logged (without sensitive details)

---

## 3. Strict Redirect Policy

**Principle:** For API calls carrying authentication credentials, disable automatic redirect following entirely. Redirects to a different domain can leak non-standard authentication headers that are not automatically stripped by most HTTP clients.

**The vulnerability:** The `Authorization: Bearer` header is stripped on cross-domain redirects by most HTTP libraries. Non-standard headers (`x-api-key`, `Ocp-Apim-Subscription-Key`, `api-key`, `x-goog-api-key`) are NOT stripped — they follow the redirect to the destination domain.

**Policy:** Set redirect policy to "none" or "error" for all calls that carry authentication headers. If a redirect is received unexpectedly, treat it as a security event and log it.

**When redirects are legitimate:** Only allow redirects on calls that carry no authentication and fetch only public resources. Even then, limit the number of allowed redirects (maximum 3) and validate that the redirect destination is in an expected domain allowlist.

**Code review checklist:**
- [ ] All API calls with auth headers have redirect following disabled
- [ ] Unexpected 3xx responses are logged as security events
- [ ] No API credentials are passed as query parameters (visible in URLs, logs, Referer)
- [ ] `Cache-Control: no-store` is set on responses containing sensitive data

---

## 4. Certificate Pinning — Concepts and Tradeoffs

**When to use pinning:** For applications that communicate with a small, known set of high-value endpoints (e.g., a single LLM API provider). Not appropriate for general-purpose HTTP clients that connect to arbitrary domains.

**Pinning types by risk profile:**

| Type | Survives key rotation | Block risk | Use case |
|---|---|---|---|
| Leaf certificate | No | High | Do not use — breaks on every renewal |
| SPKI (public key SHA-256) | Yes, if key is reused | Medium | Recommended for known stable endpoints |
| CA-level | Yes | Low | Acceptable fallback when SPKI is impractical |

**Mandatory backup pin:** Always configure a secondary pin. Without a backup pin, a key rotation at the server will lock out all users until the application is updated. The backup pin should be for a different key on the same CA or a backup CA.

**SPKI extraction:** `openssl s_client -connect host:443 | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64`

**HPKP is obsolete:** HTTP Public Key Pinning (browser header mechanism) was removed from all major browsers in 2018 due to the "pin bombing" denial-of-service attack vector.

**Risk of pinning in desktop apps:** Installing a certificate authority into the OS trust store bypasses pinning that relies on the OS truststore. Corporate proxies (Zscaler, Netskope, Palo Alto) routinely install their CA. Consider validating the certificate issuer against an expected set and alerting (not blocking) when an unexpected issuer is detected.

---

## 5. Timeout Configuration

**Principle:** All network connections must have explicit timeouts for connection establishment and total request duration. Absence of timeouts enables resource exhaustion attacks and hangs from slow or malicious servers.

**Three timeout dimensions:**
- **Connect timeout:** Maximum time to establish the TCP connection and complete the TLS handshake. Should be short (3–10 seconds).
- **Read timeout:** Maximum time to receive the response body after the request is sent. Varies by service characteristics.
- **Total request timeout:** A hard ceiling on the entire operation, preventing indefinite hangs in streaming contexts.

**Representative values by service type:**

| Service type | Connect (s) | Read (s) | Notes |
|---|---|---|---|
| Fast inference API (sub-second latency) | 3 | 30 | |
| Standard LLM API | 5 | 300 | Streaming responses can be long |
| Long-context or reasoning model | 5 | 600 | Chain-of-thought can exceed 5 minutes |
| RSS / public data fetch | 10 | 15 | Cap response size at 5 MB |
| Update server | 5 | 60 | Verify hash before writing to disk |
| Local service (no network) | 2 | 600 | No network latency; CPU-bound |

**Code review checklist:**
- [ ] Every HTTP client has a connect timeout configured
- [ ] Every HTTP client has a read/total timeout configured
- [ ] No timeout is set to zero or unlimited (check for missing timeout = system default = often unlimited)
- [ ] Response size is bounded — decompressed body has a maximum size check

---

## 6. DNS Protection

**DNS-over-HTTPS (DoH):** Standard DNS queries over UDP port 53 are unencrypted and subject to hijacking, eavesdropping, and manipulation. For applications with privacy requirements, configure DNS resolution to use DoH providers.

**Recommended DoH providers:**
- Cloudflare (`https://cloudflare-dns.com/dns-query`) — 11ms average, no-log audited by KPMG
- Quad9 (`https://dns.quad9.ch/dns-query`) — Swiss jurisdiction, GDPR-compliant
- Google (`https://dns.google/dns-query`) — low latency, logs retained

**SOCKS5 proxy DNS leak:** When using a SOCKS5 proxy, the URL scheme determines where DNS resolution occurs. `socks5://` resolves DNS locally (leak). `socks5h://` sends the hostname to the proxy for remote resolution (no leak). The missing `h` is the entire difference.

**DNS TTL floor:** When validating IP addresses for SSRF protection, enforce a minimum TTL floor (e.g., 60 seconds) on cached DNS results to prevent DNS rebinding attacks.

**Code review checklist:**
- [ ] Proxy URLs use `socks5h://`, never `socks5://`
- [ ] DNS validation results are not re-resolved between check and use (TOCTOU)
- [ ] DNS caching has a minimum TTL floor when used for security decisions

---

## 7. Rate Limiting and Request Size Limits

**Rate limiting:** Implement rate limits on both inbound API calls (from users or agents) and outbound calls (to external services). Rate limits prevent:
- Amplification attacks where one user triggers many expensive external calls
- Cost exhaustion via API quota abuse
- Credential stuffing against authentication endpoints

**Token bucket or sliding window algorithms** are preferred over fixed windows (which are vulnerable to burst attacks at window boundaries).

**Request size limits:** All endpoints that accept request bodies must enforce a maximum size before beginning to parse or process the body. Parsing can be CPU and memory intensive — a 10 GB body submitted to a JSON parser causes denial of service.

**Decompression limits:** Any endpoint that accepts compressed data (gzip, brotli, zstd) must enforce a maximum decompressed size. Compressed bombs can expand by factors of 1000:1.

**Code review checklist:**
- [ ] All inbound endpoints have a maximum request body size limit
- [ ] Decompressed body sizes are bounded
- [ ] Rate limiting is applied before expensive processing (auth, parsing, database)
- [ ] Rate limit state is stored server-side, not in client-supplied tokens

---

## 8. CORS, Connection Pooling, and Log Redaction

**CORS:** Never use `Access-Control-Allow-Origin: *` on endpoints that return sensitive data or require authentication. Maintain an explicit origin allowlist and reflect only the matched origin. `Access-Control-Allow-Origin: *` combined with `Access-Control-Allow-Credentials: true` is a credential exposure.

**Connection pool limits:** Unbounded pools are a resource exhaustion vector. Set a maximum pool size per host and a maximum idle lifetime. In multi-tenant contexts, never reuse a connection authenticated with one user's credentials for a different user's request.

**Secret redaction in logs:** API keys, tokens, and session credentials must never appear in any log output — including debug logs, error messages, and stack traces. Common leak vectors: HTTP client debug logs (print request headers), error messages containing full URLs (credential query parameters), and stack captures of local variables holding token strings. Set HTTP library log levels to WARN or above in production. Replace credential values with `[REDACTED]` before any log write.

---

## Code Review Checklist

- [ ] Minimum TLS version set explicitly (TLS 1.3 preferred) — not inherited from OS defaults
- [ ] Certificate validation enabled — no "accept all" or "skip hostname" flags in production code
- [ ] No HTTP fallback path exists when HTTPS fails
- [ ] Redirect following disabled for all calls carrying auth headers
- [ ] Unexpected 3xx responses logged as security events
- [ ] API credentials never passed as URL query parameters
- [ ] Connect timeout configured on every HTTP client
- [ ] Read/total timeout configured on every HTTP client
- [ ] Response body size bounded pre- and post-decompression
- [ ] Connection pool has an upper bound (not unlimited)
- [ ] SOCKS5 proxies use `socks5h://` scheme — not `socks5://`
- [ ] DNS validation not subject to TOCTOU rebinding (TTL floor enforced)
- [ ] Inbound endpoints enforce a maximum request body size before parsing
- [ ] Rate limiting applied before expensive processing
- [ ] Auth headers never appear in any log output
- [ ] HTTP debug logging disabled or filtered in production
