# Defense Patterns: Authentication
> Secure authentication patterns for M2M, JWT, API keys, and OAuth2 — with key rotation and code review checklist.

---

## 1. Machine-to-Machine (M2M) Authentication

When two services authenticate each other without a human in the loop, the core principle is: **the private key never leaves the machine that generated it.**

### Certificate-Based Authentication (Recommended)

**Pattern:** Each machine generates a key pair locally. A trusted Certificate Authority (CA) signs a certificate binding the public key to an identity. The certificate, not the key, is exchanged. The CA can be embedded in the application binary (no external infrastructure required).

**Ed25519 key pairs** are the current recommendation for M2M:
- 256-bit security level with 32-byte keys (compact, fast)
- Deterministic signatures (no CSPRNG required at signing time, no nonce reuse risk unlike ECDSA)
- Resistant to timing attacks by design in most implementations
- Supported natively by OpenSSH certificates, TLS 1.3 (RFC 8446), and JWT (EdDSA)

**Ephemeral certificates with short TTL:**

| Use case | Recommended TTL | Renewal trigger |
|---|---|---|
| M2M service-to-service | 1 hour | At 2/3 of TTL (with jitter) |
| Human user session | 4–8 hours | At 2/3 of TTL |
| Host / machine identity | 7 days | At 2/3 of TTL |
| Bootstrap enrollment token | 1 hour, single-use | N/A — consumed immediately |

Short TTL certificates limit the exposure window after compromise without requiring active revocation. If a certificate is compromised, it expires naturally within the TTL window — no coordination required to limit damage.

**Industry evidence:** Teleport Machine ID uses 1-hour certificates renewed every 20 minutes. Nebula (Slack, 50,000+ hosts) uses offline CA with per-node certificates. Vault SSH recommends 10-30 minutes for bots. All converge on the same model: ephemeral credentials, short windows, automatic renewal.

### Bootstrap: Solving the First-Authentication Problem

The first time a machine presents itself to a CA, it has no credential yet. The standard pattern used by Teleport, step-ca, and Nebula:

1. An operator generates a **single-use enrollment token** with a short TTL (e.g., 1 hour).
2. The new machine presents the token and its freshly generated public key to the CA.
3. The CA validates and consumes the token (marking it as used — it cannot be replayed).
4. The CA signs a certificate and returns it along with its public key (trust bundle).
5. The machine verifies the CA identity via a pre-shared fingerprint (SHA-256 of the CA public key) — this prevents MITM during bootstrap.

Without step 5 (CA fingerprint verification), an attacker positioned between the enrolling machine and the CA can substitute their own CA and issue fraudulent certificates.

---

## 2. Key Rotation Strategies

**Why rotation matters:** A compromised key that is never rotated grants permanent access. Rotation limits the exposure window to the rotation interval.

### Overlap Period (Zero-Downtime Rotation)

Never cut over instantly from old key to new key. The pattern:

1. Generate the new key/certificate.
2. Begin accepting both old and new credentials simultaneously (overlap window).
3. After all active clients have renewed (typically 1–2 TTL cycles), stop accepting the old credential.
4. Revoke or mark the old key as expired.

The overlap window should be at least one full TTL of the credential being rotated. For a certificate with a 1-hour TTL, the overlap window should be at least 1 hour — ideally 2.

**Jitter on renewal:** When hundreds of clients renew simultaneously (thundering herd), add random jitter to the renewal trigger. Formula used by step-ca: `renew_at = issue_time + (ttl * 2/3) + rand(0, ttl/20)`. This spreads renewal load across a window rather than spiking it.

### Rotation Triggers

- **Scheduled:** Fixed interval (e.g., every 30 days for long-lived keys).
- **Credential-age based:** Rotate before the key reaches a maximum age.
- **Incident-driven:** Immediate rotation on any suspected compromise.
- **Automatic on renewal:** Each certificate renewal issues a new short-lived cert — rotation is implicit and continuous.

### Graceful Degradation

If the key rotation service is unavailable, existing sessions with valid certificates must continue to function. Design the system so certificate renewal failure causes an alert, not an outage. Maintain a buffer of at least 1/3 TTL between renewal attempts and expiration.

---

## 3. JWT Best Practices

**Algorithm:**
- Use **EdDSA (Ed25519)** for asymmetric signatures — compact, fast, no nonce management.
- Use **HS256 / HS512** (HMAC-SHA) only for symmetric contexts where both parties share the secret and the secret is managed securely.
- Never accept `"alg": "none"` — explicitly reject it in token validation.
- Never allow the client to specify the algorithm — the server must hardcode the expected algorithm.

**CVE example:** CVE-2015-9235 — numerous JWT libraries accepted `"alg": "none"`, allowing tokens with no signature at all to be validated as authentic.

**Expiration:**
- Set `exp` (expiration) on every token. There is no legitimate use case for a JWT without expiration.
- Access tokens: 15 minutes to 1 hour.
- Refresh tokens: 1–30 days, depending on the application's risk tolerance.
- Reject tokens where `exp` is missing or in the past — fail closed.

**Audience validation:**
- Set `aud` (audience) in every token and validate it on the receiving service.
- A token issued for service A must be rejected by service B, even if both trust the same signing key.
- Without audience validation, a token issued for a low-privilege service can be replayed against a high-privilege service.

**Payload content:**
- Never store sensitive data in the JWT payload — the payload is base64-encoded, not encrypted. It is visible to anyone who holds the token.
- Store only the minimum identity information needed for authorization decisions (user ID, roles, scopes).
- For sensitive claims, use a JWE (JSON Web Encryption) or return them in an encrypted secondary lookup.

**Token storage on clients:**
- Prefer `HttpOnly`, `Secure`, `SameSite=Strict` cookies over localStorage. Cookies with these flags are not accessible to JavaScript, blocking XSS-based theft.
- Tokens stored in localStorage are accessible to any JavaScript on the page — a single XSS vulnerability compromises all stored tokens.

---

## 4. API Key Management

**Scoped permissions:** Every API key must have explicitly defined, minimal permissions. A key used for read-only data access must not carry write or admin scopes. Broad or wildcard scopes (`scope: "*"`) mean a single key compromise grants full account access.

**Key format:**
- Use a recognizable, scannable prefix (e.g., `svc_`, `prod_`, `sk-`). This enables automated secrets scanning in code and logs.
- Use at least 128 bits of CSPRNG entropy in the key body.
- Optionally include an HMAC checksum as a suffix to allow fast client-side format validation without a network call.

**Rate limiting per key:** Enforce rate limits individually per API key, not just per IP address. Per-IP limits are bypassable with distributed request sources. Per-key limits catch abuse regardless of origin.

**Rotation:**
- Provide a rotation API that issues a new key and supports an overlap window (old key remains valid for a configurable grace period).
- Never require zero-downtime rotation without an overlap window — this forces customers to schedule downtime.
- Alert on keys not rotated within a policy window (e.g., 90 days).

**Revocation:** Build revocation as a first-class operation. On compromise, a key must be revocable in seconds, not minutes. Revocation status must propagate to all enforcement points within one request latency.

---

## 5. OAuth2 Security

**PKCE (Proof Key for Code Exchange):**
- Mandatory for all public clients (single-page apps, mobile apps, CLIs) regardless of whether a client secret is available.
- The `code_verifier` is a CSPRNG-generated random string (43-128 characters from `[A-Za-z0-9._~-]`).
- The `code_challenge` sent in the authorization request is `BASE64URL(SHA-256(code_verifier))`.
- Without PKCE, a malicious app that intercepts the authorization code can exchange it for tokens — there is no way to distinguish the legitimate app from the attacker.

**Refresh token rotation:**
- Issue a new refresh token on every use. Invalidate the old one immediately.
- If a refresh token is used twice (once by the legitimate client, once by an attacker), the second use reveals the compromise — invalidate the entire token family (all refresh tokens for that session).
- This pattern detects replay attacks and limits the window of exposure from a stolen refresh token.

**Token binding:**
- Bind tokens to a client identifier (device ID, client fingerprint) where the threat model warrants it.
- A token bound to a specific client is useless if extracted and replayed from a different client.

**OAuth2 `state` parameter:**
- Always include a CSPRNG-generated `state` parameter in the authorization request.
- Validate it on callback to prevent CSRF attacks on the OAuth2 flow.
- The `state` must be unpredictable — a fixed or guessable value provides no protection.

**Scope minimization:**
- Request only the scopes the application actually needs.
- Avoid wildcard or omnibus scopes. A token with `scope: "*"` (as granted by Reddit script apps by default) means a single token compromise grants every permission available on the account.

**Token storage — refresh tokens:**
- Treat refresh tokens as secrets equivalent to passwords. They are long-lived and often non-expiring.
- Store in the OS keychain or equivalent, not in a database column without encryption.
- A leaked refresh token in a Git commit or log file must trigger immediate revocation.

---

## 6. Code Review Checklist

Before every release, verify each item is true across the entire codebase:

**M2M and certificates:**
- [ ] Private keys are generated locally and never transmitted — only public keys or signed certificates cross trust boundaries
- [ ] All certificates have an explicit expiration (no perpetual certificates for automated systems)
- [ ] Enrollment tokens are single-use and validated server-side for prior use before accepting
- [ ] CA fingerprint is verified during bootstrap to prevent MITM
- [ ] Certificate renewal uses jitter to prevent thundering herd

**JWT:**
- [ ] The accepted signing algorithm is hardcoded server-side — clients cannot override it
- [ ] `"alg": "none"` is explicitly rejected
- [ ] Every issued token has an `exp` claim; tokens without `exp` are rejected
- [ ] `aud` (audience) is set and validated on every token
- [ ] JWT payload contains no sensitive data (no passwords, no full account details)

**API keys:**
- [ ] Every key has explicit, minimal, named scopes — no wildcard scopes
- [ ] Keys are generated with at least 128 bits of CSPRNG entropy
- [ ] Per-key rate limits are enforced independently of IP-based limits
- [ ] A revocation path exists that takes effect within one request cycle
- [ ] Key rotation is supported with an overlap window

**OAuth2:**
- [ ] PKCE is used for all authorization code flows in public clients
- [ ] `state` parameter is a CSPRNG value, validated on callback
- [ ] Refresh token rotation is implemented — each refresh issues a new token
- [ ] Requested scopes are the minimum necessary
- [ ] Refresh tokens are stored with the same protection as passwords (encrypted, not in plaintext columns)
