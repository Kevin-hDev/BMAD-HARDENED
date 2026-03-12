# Defense Patterns: Application Framework Hardening
> Secure configuration and coding patterns for web, desktop, and mobile application frameworks: CSP, CORS, permissions, sanitization, and WebView hardening.

---

## Overview

Application frameworks expose multiple attack surfaces: content injection via rendered HTML, overly permissive cross-origin policies, missing authentication on privileged routes or commands, and insecure defaults for WebView features. The patterns below apply to any framework that renders web content (browser, desktop WebView, mobile WebView) and to any backend command dispatch system.

---

## DEF-FRW-01 — Content Security Policy (CSP)

**Pattern:** A strict CSP is declared at the framework level, not in application code. It restricts which sources can execute scripts, load styles, connect to endpoints, and embed frames.

**Minimum secure directives:**

| Directive | Recommended value | Rationale |
|---|---|---|
| `default-src` | `'self'` | Deny all unspecified sources |
| `script-src` | `'self'` | No `'unsafe-inline'` or `'unsafe-eval'` in production |
| `object-src` | `'none'` | Block plugins (Flash, Java, PDF viewers) |
| `frame-src` | `'none'` | Block all iframes unless explicitly needed |
| `frame-ancestors` | `'none'` | Block clickjacking via framing by external pages |
| `form-action` | `'self'` | Block form exfiltration to external destinations |
| `base-uri` | `'self'` | Block `<base>` tag hijacking |
| `connect-src` | Explicit allowlist | Block exfiltration to unexpected endpoints |

**Key risks from weak CSP:**
- `'unsafe-eval'` enables execution of strings as code (eval, Function constructor, setTimeout with string)
- `'unsafe-inline'` enables inline script execution (XSS payloads injected via markup)
- Missing `object-src 'none'` allows plugin-based execution
- Missing `frame-ancestors 'none'` allows clickjacking

**Development vs. production:** Development builds may need relaxed CSP (HMR WebSocket, inline source maps). Use separate dev CSP configuration — never ship development CSP to production.

**Code review checklist:**
- [ ] CSP is set at the framework/server level, not only via meta tags (meta tags cannot set `frame-ancestors`)
- [ ] No `'unsafe-eval'` in production script-src
- [ ] No `'unsafe-inline'` in production script-src (use nonces or hashes for legitimate inline scripts)
- [ ] `object-src 'none'` is present
- [ ] `frame-ancestors 'none'` is present (or restricted to a specific known parent)

---

## DEF-FRW-02 — CORS Configuration

**Pattern:** Cross-Origin Resource Sharing is configured with an explicit allowlist of trusted origins. Reflecting the request's `Origin` header back as `Access-Control-Allow-Origin` is equivalent to disabling CORS.

**Rules:**
- Never use `Access-Control-Allow-Origin: *` on endpoints that handle authenticated requests
- Never reflect `Origin` dynamically without validating against an allowlist first
- `Access-Control-Allow-Credentials: true` combined with a wildcard or reflected origin is a critical vulnerability (allows cross-origin session riding)
- Preflight cache (`Access-Control-Max-Age`) should be set to reduce preflight latency without being excessively long

**Applies to:** REST APIs, GraphQL endpoints, WebSocket upgrade endpoints, and any endpoint that handles browser-originated requests.

**Code review checklist:**
- [ ] `ACAO` header value comes from a static allowlist, not from the request `Origin` header
- [ ] `ACAO: *` is not used on any authenticated endpoint
- [ ] `Access-Control-Allow-Credentials: true` is not combined with a non-specific origin
- [ ] CORS configuration is centralized (not scattered per-route)

---

## DEF-FRW-03 — Deny-by-Default Permission Model

**Pattern:** The framework's permission/capability system starts with everything denied. Permissions are granted explicitly, scoped to the minimum required, and reviewed before each change.

**Key properties:**
- A new feature does not inherit any existing permission — it starts with none
- Permissions are scoped to specific callers (window label, user role, authenticated session)
- Wildcard grants (`*`, `**`) are forbidden in production configurations
- Sensitive permissions (filesystem write, process execution, secrets access) require explicit, separate grants

**Applies to:** Desktop framework capabilities, mobile platform permissions, server-side authorization middleware, API gateway policies.

**Code review checklist:**
- [ ] No wildcard grants for sensitive operations
- [ ] Each permission grant has a documented, reviewed justification
- [ ] New routes/commands default to requiring authentication unless explicitly public
- [ ] CI pipeline flags wildcard or overly broad permission changes

---

## DEF-FRW-04 — Authentication Guards on Routes and Commands

**Pattern:** Every route or backend command that performs a privileged operation checks authentication and authorization before executing. The check happens inside the handler, not only in middleware (defense in depth).

**Guard checklist per handler:**
1. Is the caller authenticated (valid session/token)?
2. Does the caller have the required role or permission for this specific operation?
3. Is the caller's session still valid (not expired)?
4. Is the caller's origin/window/context the expected one (not an iframe or unexpected window)?

**Fail-closed principle:** If any check fails, the handler returns an error immediately and does not fall through to execute the privileged operation.

**Code review checklist:**
- [ ] Every sensitive command/route begins with an authentication check
- [ ] Authorization is checked separately from authentication (authn != authz)
- [ ] Session expiry is enforced — a long-lived session does not grant permanent access
- [ ] Error responses do not reveal whether the failure was authn or authz (prevents enumeration)

---

## DEF-FRW-05 — HTML Sanitization for User-Generated Content

**Pattern:** Any HTML content from an external source (user input, scraped content, LLM output, API response) is sanitized through a library that parses and rebuilds the HTML against an explicit allowlist before rendering.

**Important properties:**
- Sanitization happens server-side or in a trusted process, not only in the browser (defense in depth)
- The sanitizer uses an allowlist of tags and attributes — not a blocklist (blocklists miss mutation XSS variants)
- SVG and MathML are explicitly blocked unless required (both are sources of mutation XSS)
- `data-*` attributes are blocked by default (CSS exfiltration vector)
- Event handler attributes (`on*`) are always blocked
- `href` and `src` attributes are protocol-filtered: only `http:`, `https:`, `mailto:` are allowed

**Real incident:** CVE-2025-26791 (DOMPurify < 3.2.4, February 2025) — MathML/SVG namespace confusion allowed sanitization bypass via mutation XSS. Upgrade to DOMPurify >= 3.2.4.

**Alternative for structured content (e.g., LLM output):** Use a Markdown-to-AST parser with a sanitizing rehype/remark pipeline instead of `innerHTML`. AST-based pipelines cannot produce mutation XSS because they never go through a re-parse cycle.

**Code review checklist:**
- [ ] `dangerouslySetInnerHTML`, `innerHTML`, `v-html`, or equivalent is only used after sanitization
- [ ] Sanitization uses an allowlist library, not a regex or manual blocklist
- [ ] SVG and MathML are explicitly disallowed unless required
- [ ] `href`/`src` attributes are protocol-filtered after sanitization
- [ ] Sanitization library is pinned to a version without known CVEs

---

## DEF-FRW-06 — Secure WebView Defaults

**Pattern:** WebView instances are configured to disable dangerous features that are enabled by default in general-purpose browsers but not needed in controlled desktop or mobile applications.

**Features to disable:**

| Feature | Risk if enabled | How to disable |
|---|---|---|
| File drag-and-drop | Drops can trigger file:// URIs or bypass CSP | Set `fileDropEnabled: false` |
| Developer tools | Full DOM and memory access for anyone who opens them | Disable in production builds via feature flags |
| Global bridge access | Exposes IPC internals as a global window variable to all scripts | Set `withGlobalTauri: false` or equivalent |
| Navigation to arbitrary URLs | Allows renderer to navigate to attacker-controlled content | Restrict allowed navigation origins |
| Autoplay | Not a security issue, but affects XSS payloads using audio/video events | Restrict if not needed |
| Popups | Can open new windows outside CSP scope | Block unless explicitly needed |

**Code review checklist:**
- [ ] File drop is disabled on all WebView windows
- [ ] Developer tools are disabled via build feature flag (not config alone)
- [ ] Global bridge/internals access is disabled in production
- [ ] WebView is not allowed to navigate to arbitrary external URLs
- [ ] DevTools feature flag is not included in release builds

---

## DEF-FRW-07 — iframe Sandboxing and Isolation

**Pattern:** Any iframe embedded in the application is sandboxed to the minimum permissions it needs. Sensitive operations are never accessible from an iframe context.

**Sandbox attribute permissions — grant only what is required:**
- `allow-scripts`: grants JavaScript execution — omit if not needed
- `allow-same-origin`: grants access to parent's origin — almost never needed in a true sandbox
- `allow-same-origin` + `allow-scripts` together = sandbox bypass (grants full same-origin access to scripts)

**IPC isolation pattern:** A sandboxed iframe can serve as a verification layer that intercepts and filters IPC messages before they reach the privileged backend. The iframe's script runs in a separate context from the main application, preventing supply-chain attacks in npm dependencies from accessing privileged IPC.

**Limitation:** The isolation iframe is a defense-in-depth measure, not an absolute boundary. An XSS in the main application window can bypass the isolation layer if the isolation key is accessible to that window's JavaScript context.

**Code review checklist:**
- [ ] All iframes use the `sandbox` attribute
- [ ] `allow-same-origin` + `allow-scripts` are never combined in a sandbox
- [ ] IPC commands are filtered in the isolation layer against a static allowlist
- [ ] The isolation key is not marked as extractable by JavaScript

---

## DEF-FRW-08 — Subresource Integrity (SRI)

**Pattern:** External scripts and stylesheets loaded from CDNs or third-party origins include an `integrity` attribute containing a hash of the expected content. The browser refuses to execute content that does not match the hash.

**Applies to:** `<script src="...">` and `<link rel="stylesheet" href="...">` tags pointing to any origin other than `'self'`.

**Key property:** SRI does not protect against a compromised CDN that serves the malicious file AND updates the SRI hash in the application. SRI is most effective combined with a strict CSP that limits which origins can serve scripts.

**Code review checklist:**
- [ ] All external script and stylesheet tags include `integrity` and `crossorigin="anonymous"`
- [ ] SRI hashes are generated from the production version of the file (not development builds)
- [ ] A dependency update process re-generates SRI hashes after each upgrade

---

## DEF-FRW-09 — Build Configuration: No Secrets in Bundled Assets

**Pattern:** Build tool environment variable prefixes are configured to include only the explicitly intended prefix. A misconfiguration that exposes all environment variables to the bundle can leak signing keys, API keys, and internal configuration.

**Code review checklist:**
- [ ] Bundler env prefix is set to the minimum necessary (e.g., only `VITE_` or `NEXT_PUBLIC_`)
- [ ] The CI build log does not echo secret environment variables
- [ ] The production bundle is scanned for known secret patterns as part of CI (no `sk-`, `-----BEGIN`, `AKIA`, etc.)

