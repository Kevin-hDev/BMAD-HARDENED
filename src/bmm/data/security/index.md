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

## The Mask — Offensive Techniques (the-mask/)

> Charge UNIQUEMENT quand The Mask est actif (Party Mode / Red Team vs Blue Team Elicitation #17).
> Activation : REACTIVE — pas de workflow dédié.
> Techniques offensives avec PoCs. INDEX_THEN_SELECTIVE : charger 1-2 fichiers max selon le contexte.

| File | Tags | Summary |
|------|------|---------|
| the-mask/atk-exploit-web.md | exploit, offensive, web, input, api, ssrf | SSRF→IMDS, SQLi blind/second-order, XSS (stored/DOM/mXSS), CSRF bypass, deserialization gadget chains |
| the-mask/atk-exploit-desktop.md | exploit, offensive, desktop, ipc, binary, red-team | DLL hijacking, IPC abuse (pipes/D-Bus/XPC), sandbox escape Electron/Tauri, UAC/polkit/TCC bypass |
| the-mask/atk-exploit-llm.md | exploit, offensive, llm, ai-agent, mcp, prompt-injection | Prompt injection directe/indirecte (XPIA), jailbreak patterns, exfil via tool use, MCP abuse |
| the-mask/atk-exploit-mobile.md | exploit, offensive, mobile, binary, pentest | Cert pinning bypass (Frida/objection), deep link hijacking, WebView bridge RCE, keystore/keychain extraction |
| the-mask/atk-exploit-chains.md | exploit, offensive, chain-attack, architecture, all | Chaînes multi-étapes : SSRF→IMDS→RCE, XSS→Admin→RCE, supply chain→CI/CD, prompt injection→exfil |
| the-mask/atk-exploit-social.md | exploit, offensive, social-engineering, phishing, pentest | Spear phishing, MFA bypass EvilGinx, pretexting, typosquatting, OAuth consent phishing, credential stuffing |

---

## Platform-Specific (platform/)

> Charger 1 seul fichier selon l'OS cible du projet.

| File | Tags | Summary |
|------|------|---------|
| platform/platform-windows.md | windows, desktop, os | UAC bypass, DLL hijack, DPAPI, WDAC, AppContainer, WebView2 |
| platform/platform-linux.md | linux, server, os | SELinux, seccomp, Landlock, namespaces, polkit, eBPF, D-Bus |
| platform/platform-macos.md | macos, desktop, os | TCC, Gatekeeper, SIP, Hardened Runtime, dylib, XPC, Keychain |
| platform/platform-ios.md | ios, mobile | Keychain, ATS, cert pinning, jailbreak, Frida, Universal Links |
| platform/platform-android.md | android, mobile | KeyStore, Network Security Config, cert pinning, root, intents |

## Stack-Specific (stack/)

> Charger 1 seul fichier selon la stack detectee (cf. build-detection.md).

| File | Tags | Summary |
|------|------|---------|
| stack/stack-python.md | python, django, flask, fastapi | ORM injection, SSTI, pickle, Werkzeug, PyPI supply chain |
| stack/stack-js-node.md | javascript, node, express, nextjs, nestjs | Prototype pollution, React2Shell, middleware bypass, RSC |
| stack/stack-php.md | php, laravel, livewire | PHPGGC, mass assignment, APP_KEY, Blade, Livewire, type juggling |
| stack/stack-frontend.md | vue, nuxt, angular, svelte, sveltekit, vite, spa, frontend | v-html XSS, SSR leaks, supply chain SPA, CSP, Trusted Types |

## Infrastructure (infra/)

> Charger 1 fichier selon l'infra du projet.

| File | Tags | Summary |
|------|------|---------|
| infra/infra-containers.md | docker, kubernetes, container, isolation | runc escape, RBAC, Pod security, IngressNightmare |
| infra/infra-cicd.md | github-actions, gitlab, ci-cd, pipeline, supply-chain | Expression injection, runners, tj-actions, SLSA |
| infra/infra-cloud.md | aws, gcp, azure, cloud, serverless, iam | SSRF→IMDS, IAM, Lambda injection, secrets management |

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
| Windows desktop | windows, desktop |
| Linux server | linux, server |
| macOS desktop | macos, desktop |
| iOS mobile | ios, mobile |
| Android mobile | android, mobile |
| Python backend | python + framework name |
| Node.js backend | javascript, node |
| PHP backend | php, laravel |
| Vue/Angular/Svelte SPA | frontend, spa |
| Docker / Kubernetes | container, docker, kubernetes |
| CI/CD pipelines | ci-cd, pipeline |
| AWS / GCP / Azure | cloud + provider name |
