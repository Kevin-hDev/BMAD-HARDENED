# Attack Patterns: Network, TLS, and API Security

> Universal attack patterns targeting TLS configuration, HTTP redirects, DNS, proxy interception, and API surface — applicable to any application stack.

---

## 1. TLS Downgrade Attack

**Description:** An attacker with a man-in-the-middle network position intercepts the TLS ClientHello and forces negotiation to an older protocol version (TLS 1.2 or below), enabling exploitation of known cipher weaknesses.

**Vector:** Requires a MITM position — corporate proxy, rogue Wi-Fi, BGP hijacking, or DNS poisoning.

**Impact:** Traffic decryption if weak cipher suites are accepted (3DES, RC4, CBC modes). Enables BEAST, POODLE, and Lucky13 attacks against TLS 1.2 CBC.

**What to look for in code review:**
- Absence of a minimum TLS version floor (TLS 1.3 minimum recommended)
- Use of a system-level TLS backend without explicit version constraints
- Any configuration that accepts TLS 1.0 or 1.1

---

## 2. API Key Leak via HTTP Redirect

**Description:** Non-standard authentication headers (custom `x-api-key`, vendor-specific subscription key headers) are not automatically stripped by most HTTP clients when following cross-domain redirects. Standard `Authorization: Bearer` headers are usually stripped; custom headers are not.

**Vector:** Passive — the target application compromises itself. Triggered by DNS poisoning, BGP hijacking, or a compromised upstream endpoint returning a 301/302 redirect to an attacker-controlled domain.

**Impact:** Silent exfiltration of API credentials. The attacker receives the key as a normal HTTP header in the redirect target request. Leads to unauthorized usage charges, data access, and session takeover.

**Real examples:** Anthropic `x-api-key`, Azure `Ocp-Apim-Subscription-Key`, Azure OpenAI `api-key`, Google `x-goog-api-key` — all non-standard headers that bypass automatic strip logic in common HTTP libraries.

**What to look for in code review:**
- Any HTTP call carrying a non-standard auth header that also allows redirects
- Absence of explicit redirect policy set to "none" or "error on redirect"
- API tokens passed as query parameters (visible in server logs, Referer headers, URLs)

---

## 3. TLS / HTTP Fingerprinting and Bot Detection

**Description:** Anti-bot systems (Cloudflare, AWS WAF, Akamai, DataDome) fingerprint HTTP clients at the TLS handshake layer (JA3/JA4+) and at the HTTP/2 frame layer (SETTINGS frame parameters). A client whose fingerprint does not match a known browser profile is blocked or served degraded content — before any HTTP headers are examined.

**Vector:** The application's HTTP library identity is detected at the TCP/TLS level.

**Impact (for scraping/automation use cases):** Silent blocking, CAPTCHA challenges, or honeypot content. The attack is asymmetric — the attacker learns the application's true behavior while the application receives false data.

**JA4+ (FoxIO, 2025):** Successor to JA3. Produces a readable hash by sorting cipher suites and extensions before hashing, making it resistant to Chrome's randomization (v110+). AWS WAF added JA4 support in March 2025.

**HTTP/2 SETTINGS fingerprint:** Chrome sends `INITIAL_WINDOW_SIZE: 6291456` plus a WINDOW_UPDATE of 15663105. A default RFC-compliant HTTP client sends 65535 — a 96x discrepancy that is immediately detectable.

**What to look for in code review:**
- Use of HTTP libraries with known non-browser TLS fingerprints for traffic that must appear browser-originated
- Absence of ALPN negotiation including `h2`
- HTTP/2 SETTINGS values left at RFC defaults when browser impersonation is required

---

## 4. CVE-2025-66418 — Unlimited Decompression DoS

**CVSS: 8.9 (High)**
**Affects:** urllib3 < 2.6.0 (Python HTTP library used by the `requests` ecosystem)
**Fixed in:** urllib3 2.6.0

**Description:** No limit on decompressed response body size. A malicious or compromised server returns a compressed payload (gzip, deflate, brotli) of a few kilobytes that expands to multiple gigabytes in memory — a "zip bomb" attack.

**Vector:** Any server endpoint the application queries. Attacker must control or compromise one upstream dependency.

**Impact:** Process OOM-kill, denial of service, potential data loss from abrupt termination. Related CVEs in the same release: CVE-2025-66471 (streaming resource exhaustion), CVE-2025-50181 and CVE-2025-50182 (redirect control bypass).

**What to look for:** Absence of `max_size` or equivalent constraint on decompressed response bodies. Unversioned or loosely versioned HTTP library dependencies.

---

## 5. DNS Leak via SOCKS5 Proxy

**Description:** The `socks5://` URL scheme resolves DNS locally (on the client) before sending only the IP address to the proxy. This exposes every domain contacted to the local resolver and upstream ISP, even when all HTTP content is encrypted.

**Vector:** Any application using a SOCKS5 proxy for anonymization or privacy. The correct scheme is `socks5h://` (hostname-resolution at proxy).

**Impact:** Complete mapping of APIs called, scraping behavior, and identity correlation — even though the payload is encrypted.

**What to look for in code review:**
- `socks5://` anywhere in proxy configuration (the missing `h` is the indicator)
- Note: even `socks5h://` may leak DNS on some OS/runtime combinations — verify with packet capture after configuration changes

---

## 6. MITM via Unauthenticated or Malicious Proxy

**Description:** Proxy credentials transmitted in `Proxy-Authorization` headers (HTTP CONNECT) and SOCKS5 authentication sub-protocol (RFC 1929) travel in cleartext before the TLS tunnel is established. A malicious proxy can inspect connection metadata and, if its CA certificate is installed in the OS trust store, perform a transparent MITM.

**Indicators of proxy MITM:** The TLS certificate issuer field shows a corporate proxy CA ("Zscaler Root CA", "Netskope", "Blue Coat", "Forcepoint") instead of the legitimate CA (DigiCert, Let's Encrypt).

**WebRTC IP leak (browser/Electron contexts):** WebRTC uses STUN/TURN over UDP, which bypasses proxy tunnels entirely, revealing the real IP address. Specific browser flags are required to disable non-proxied UDP.

**What to look for in code review:**
- Absence of certificate issuer validation for critical endpoints
- WebRTC or STUN not explicitly disabled in Electron/Chromium-embedded contexts
- Proxy credentials stored in plaintext configuration files

---

## 7. Certificate Validation Bypass and Pinning Bypass

**Description:** Disabling TLS certificate validation entirely ("accept any certificate") is the most severe form. Certificate pinning bypass techniques vary by pinning type:

- **Leaf certificate pinning:** Bypassed when the server renews its certificate (common with DV/Let's Encrypt).
- **SPKI pinning (public key hash):** Survives renewal if the server reuses the same key pair.
- **CA-level pinning:** Bypassed if any certificate signed by that CA is accepted.
- **OS truststore attack:** Installing a CA into the operating system trust store bypasses all pinning that relies on the OS truststore for root validation.

**HPKP (HTTP Public Key Pinning):** Obsolete since 2018. Historical attack: "pin bombing" — an attacker could pin their own CA, causing a permanent denial of service for the legitimate site.

**What to look for in code review:**
- Any flag that disables certificate verification globally or per-host
- Any flag that disables hostname verification
- Fallback to HTTP if TLS fails
- Certificate pinning without a backup/secondary pin (lock-out risk on key rotation)

---

## 8. Server-Side Request Forgery (SSRF)

**Description:** An attacker controls a URL or hostname that the application fetches on the server side, causing requests to internal network resources, cloud metadata endpoints (AWS `169.254.169.254`, GCP metadata server), or localhost services.

**Vector:** Any user-supplied or externally-supplied URL that the application fetches. Also triggered by malicious content in RSS feeds, webhooks, or API responses containing embedded URLs.

**Impact:** Access to internal services, cloud instance metadata (IAM credentials), and lateral movement within the deployment environment.

**What to look for in code review:**
- URL inputs that are not validated against an allowlist of domains/IP ranges
- Absence of blocklist for loopback (`127.0.0.0/8`), link-local (`169.254.0.0/16`), and private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`)
- DNS re-resolution after initial validation (TOCTOU — see DNS Rebinding below)

---

## 9. DNS Rebinding

**Description:** An attacker controls a DNS record with a very short TTL. The initial DNS resolution returns a legitimate IP that passes IP allowlist checks. After the TTL expires and the connection is re-established, the DNS record is changed to point to an internal IP — bypassing the original validation.

**Vector:** Any validation that checks the resolved IP at request initiation but re-resolves DNS on retry or connection reuse.

**Impact:** Bypass of IP-based SSRF protections. Access to internal resources that were supposed to be protected by the allowlist check.

**What to look for in code review:**
- IP allowlist/denylist checks performed at request build time, not enforced at the network layer
- Absence of DNS caching with a minimum TTL floor
- HTTP client with connection pooling that can reuse connections to different resolved IPs

---

## 10. HTTP Request Smuggling

**Description:** Discrepancies in how a front-end proxy and back-end server parse `Content-Length` and `Transfer-Encoding: chunked` headers allow an attacker to inject a partial request that is prepended to the next legitimate user's request.

**Vector:** Applications behind load balancers, reverse proxies, or CDNs with different HTTP parsing implementations.

**Impact:** Cache poisoning, session hijacking, bypassing access controls, and injecting malicious responses for other users.

**What to look for in code review:**
- Inconsistent HTTP/1.1 header handling across proxy and application layers
- Absence of `Transfer-Encoding: chunked` normalization at the ingress layer
- Mixed HTTP/1.1 and HTTP/2 protocol handling without proper de-multiplexing

---

## CVE and Advisory Summary

| Identifier | Score | Component | Type |
|---|---|---|---|
| CVE-2025-66418 | 8.9 | urllib3 < 2.6.0 | DoS via unlimited decompression |
| CVE-2025-66471 | — | urllib3 < 2.6.0 | DoS via streaming resource exhaustion |
| CVE-2025-50181/82 | — | urllib3 < 2.5.0 | Redirect control bypass |
| RUSTSEC-2025-0004 | Critical | Rust `openssl` crate (native-tls backend) | Use-after-free in TLS processing |
| CVE-2026-21852 | — | Sandbox environment variable injection | API key exfiltration via `ANTHROPIC_BASE_URL` override |
