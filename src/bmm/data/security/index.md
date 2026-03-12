# Security Data Index

> Read THIS file first, then load ONLY the files whose tags match the current context.
> NEVER load all files at once — select 3-5 max based on relevance.

## How to Select Files

1. Identify what the story/code touches (auth, network, LLM, etc.)
2. Match against the **Tags** column below
3. Load ONLY matching files (prefer `def-` files for code review, `atk-` for threat modeling)
4. Load `ref-` files only when you need cross-referencing or CVE lookup

---

## Attack Patterns (atk-)

| File | Tags | Summary |
|------|------|---------|
| atk-llm-prompt-injection.md | llm, ai-agent, mcp | XPIA, Policy Puppetry, jailbreaks, multi-agent injection |
| atk-llm-deception.md | llm, ai-agent | Exfiltration, sleeper agents, sandbox escape, scheming |
| atk-injection-input.md | input, database, web | SQL injection, XSS (stored/DOM/mutation), template injection |
| atk-command-injection.md | process, input, os | Shell injection, argument injection, path traversal, deserialization |
| atk-supply-chain.md | supply-chain, dependencies | Typosquatting, slopsquatting, dependency confusion, lockfile attacks |
| atk-credentials-secrets.md | secrets, auth, crypto | Memory scraping, token theft, key extraction, timing attacks |
| atk-network-tls.md | network, api, tls | TLS downgrade, SSRF, DNS rebinding, fingerprinting |
| atk-privilege-escalation.md | os, isolation, process | polkit/UAC/TCC bypass, BYOVD, container escape |
| atk-ipc-process.md | ipc, process, desktop | IPC bypass, message forgery, replay, sidecar injection |
| atk-chain-patterns.md | architecture, all | Multi-step attack chains, pivot patterns, kill chain mapping |
| atk-reverse-engineering.md | binary, desktop, mobile | Binary extraction, debug attachment, API hooking, canary detection |

## Defense Patterns (def-)

| File | Tags | Summary |
|------|------|---------|
| def-llm-pipeline.md | llm, ai-agent, mcp | 6-layer defense pipeline, Spotlighting, canary tokens |
| def-crypto-secrets.md | crypto, secrets, auth | Keychain, encryption, zeroization, CSPRNG, constant-time |
| def-auth-patterns.md | auth, api, crypto | M2M auth, key rotation, JWT, OAuth2 PKCE, API keys |
| def-network-tls.md | network, api, tls | TLS 1.3, cert pinning, timeouts, rate limiting, CORS |
| def-os-isolation.md | os, isolation, process | Least privilege, double-layering, namespaces, sandboxing |
| def-ipc-hardening.md | ipc, process, desktop | HMAC auth, anti-replay, schema validation, socket security |
| def-framework-hardening.md | web, desktop, framework | CSP, CORS, permissions, HTML sanitization, WebView security |
| def-runtime-memory.md | runtime, crypto, all | Constant-time, zeroization, bounded collections, CSPRNG, fail-closed |
| def-deception-monitoring.md | monitoring, all | Canary files/tokens, honeypots, hash-chained logs, tripwires |
| def-audit-logging.md | monitoring, auth, all | JSONL, hash chaining, what to log/never log, tamper evidence |

## Reference Files (ref-)

| File | Tags | Summary |
|------|------|---------|
| ref-agent-threat-model.md | ai-agent, llm, architecture | Lethal trifecta, MAESTRO, kill switch, behavioral drift |
| ref-cross-validation-matrix.md | all, architecture | Attack-to-defense mapping, gap analysis template |
| ref-cve-catalog.md | all | Notable CVEs by category (2019-2026), lessons learned |

---

## Tag Quick Reference

| If the code touches... | Load files tagged with... |
|------------------------|--------------------------|
| LLM / AI features | llm, ai-agent |
| User input / forms | input, web |
| Database / ORM | database, input |
| Authentication | auth, crypto, secrets |
| API endpoints | api, network, auth |
| File system / paths | os, process, input |
| Subprocess / exec | process, os |
| Dependencies / packages | supply-chain |
| Network / HTTP calls | network, tls, api |
| Desktop application | desktop, ipc, binary |
| Logging / auditing | monitoring |
| Containers / sandboxing | isolation, os |
