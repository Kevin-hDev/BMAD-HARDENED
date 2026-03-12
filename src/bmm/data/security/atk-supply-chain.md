# atk-supply-chain.md
> Attack patterns targeting the software dependency ecosystem: malicious packages, compromised maintainers, lockfile manipulation, and CI/CD pipeline exploitation.

---

## Context: Scale of the Threat (2025–2026)

| Metric | Value |
|---|---|
| npm malicious packages published in 2025 | 454,648 (Sonatype) |
| npm incident rate increase | +700% year-over-year |
| Packages compromised by Shai-Hulud worm (Sep 2025) | 500+ (v1), 25,000+ repos (v2) |
| Weekly downloads affected by chalk+debug hijack | ~2.6 billion |
| Developer accounts compromised in S1ngularity campaign | 1,000+ |

OWASP Top 10:2025 places Software Supply Chain Failures at A03:2025. The EU Cyber Resilience Act exempts non-commercial OSS — most widely used transitive dependencies remain outside regulatory scope even after mandatory reporting begins in September 2026.

---

## 1. Typosquatting

**Description:** An attacker registers a package name that closely resembles a popular legitimate package. Developers who mistype the name, or tools that auto-complete incorrectly, install the malicious version. The fake package often provides minimal functionality to avoid suspicion.

**Common name mutation patterns:**
- Character transposition: `lodash` → `lodahs`, `react` → `raect`
- Homoglyphs or digit substitution: `l0dash`, `tok1o`
- Added/removed hyphens or underscores: `fastlog` → `fast-log`, `faster_log`
- Suffix addition: `express-js`, `react-dom-js`
- Plural forms: `expresss`

**Registries affected:** npm, PyPI, crates.io, RubyGems, Go modules (via path manipulation).

**Real incidents:**
- `faster_log` (crates.io, May 2025): typosquat of `fast_log`. At runtime, extracted Ethereum and Solana private keys from environment variables and exfiltrated them. Removed September 2025.
- `evm-units`, `uniswap-utils` (crates.io, April–December 2025): plausible names for Web3 developers. Malicious `build.rs` script downloaded and executed a stage-2 payload at compile time.
- `sisaws`, `secmeasure` (PyPI, August 2025, "SilentSync"): names mimicking security tools, delivered a Remote Access Trojan stealing browser credentials.

**Code review signs:**
- Dependency list contains packages with low download counts, very recent creation dates, or single maintainers.
- Package names differ from the canonical version by one character, a hyphen, or a digit.
- No process for verifying package identity against a reference list before adding new dependencies.

---

## 2. Slopsquatting (LLM-Hallucinated Package Names)

**Description:** Language models generating code frequently hallucinate package names that do not exist. Attackers monitor LLM outputs, identify consistently hallucinated names, register those names on public registries, and wait for developers to blindly `npm install` or `pip install` what the model suggested.

**Scale:** Spracklen et al. (UT San Antonio, 2024–2025) tested 16 LLMs on 756,000 code samples. 19.7% of 2.23 million samples referenced hallucinated packages; 43% of hallucinated names appeared consistently across 10 independent queries.

**Real incidents:**
- `huggingface-cli` (PyPI): researcher Bar Lanyado (Lasso Security) registered a hallucinated name before attackers and received 30,000 authentic downloads in 3 months — proving the attack is viable at scale.
- `react-codeshift` (npm, January 2026): hallucinated package name propagated into 237 GitHub repositories via AI-generated "Agent Skills" files.
- In 2025, Aikido Security researcher Charlie Eriksen pre-registered several consistently hallucinated npm package names to prevent exploitation.

**Code review signs:**
- New dependency with no prior history in the codebase, added in a commit that was AI-assisted.
- Package name does not appear in the official documentation of the framework it supposedly extends.
- Zero stars, zero open issues, and package published within the last few weeks.

---

## 3. Dependency Confusion (Internal vs. Public Registry)

**Description:** Organizations use internal package registries for private libraries. If a package manager is configured to check both internal and public registries, an attacker can publish a package on the public registry using the same name as an internal package but with a higher version number. The package manager installs the public (malicious) version.

**Vector:** CI/CD pipelines or developer machines configured with both a private registry and a fallback to a public registry, without scoping internal package names to the private registry.

**How attackers find internal package names:**
- GitHub public repositories leaking `package.json`, `Cargo.toml`, or `requirements.txt`
- Job postings mentioning internal library names
- Binary analysis revealing import paths
- Public CI/CD logs

**Impact:** Malicious code executes on developer machines and CI/CD servers — typically during install, build, or test. Credentials, tokens, and environment variables are the primary exfiltration targets.

**Code review signs:**
- Internal package names (e.g., `@company/internal-lib`) not scoped to a specific private registry in `.npmrc`, `.cargo/config.toml`, or `pip.conf`.
- No placeholder packages published on the public registry to block name squatting.
- CI/CD configuration that references both a private and a public registry without explicit package-to-registry mapping.

---

## 4. Compromised Maintainer Account

**Description:** Rather than creating a fake package, the attacker takes over the legitimate maintainer's account and publishes a malicious version of the real package. All existing users automatically receive the backdoored version on their next update.

**Techniques used to compromise accounts:**
- Credential stuffing with leaked passwords from unrelated breaches
- Phishing: spoofed registry support emails capturing password and TOTP simultaneously via real-time proxy (evilginx, modlishka)
- Social engineering: impersonating a registry employee

**Real incidents:**
- `chalk` + `debug` + 16 other packages (npm, September 8, 2025): Attacker phished maintainer "Qix" via email from `support@npmjs.help` (not `npmjs.com`). Real-time TOTP interception gave account control. A crypto-clipper using Levenshtein distance to replace wallet addresses was injected. ~2.6 billion weekly downloads affected. Malicious versions live for ~2 hours. CISA issued an alert September 23, 2025.
- `nx` package (npm, August 26, 2025, "S1ngularity"): Maintainer account compromised. First documented supply chain attack using integrated AI CLI tools (Claude, Gemini, Q) to automate post-compromise reconnaissance and exfiltration. 1,000+ developer accounts subsequently compromised. Stolen data: GitHub tokens, npm tokens, SSH keys, `.env` files, crypto wallets.

**Code review signs:**
- Package version pinned to a floating range (e.g., `^4.0.0`) that automatically upgrades.
- No review process triggered when a dependency version changes in the lockfile.
- Automated merging (Dependabot, Renovate with `automerge: true`) for patch-level updates to packages without integrity checks.

---

## 5. Malicious Install Scripts (postinstall / preinstall Hooks)

**Description:** Package managers execute lifecycle scripts automatically during install. An attacker can embed arbitrary commands in `preinstall`, `postinstall`, or `install` script fields of a `package.json` (or the Cargo equivalent: `build.rs`). These scripts run with the developer's or CI/CD agent's full privileges.

**Real incident — Shai-Hulud worm v2 (npm, November 2025):** Used `preinstall` (harder to detect than `postinstall`) to execute a Node.js one-liner that fetched and executed a shell payload. On failure, attempted to destroy the user's home directory. Started from the compromise of `@ctrl/tinycolor` (8M+ monthly downloads), then self-replicated by stealing npm tokens and publishing malicious versions of each victim's other packages. Over 25,000 repositories contaminated.

**Typical malicious pattern:**
```
"scripts": { "preinstall": "node -e \"...fetch payload and execute...\"" }
```
Or in a Cargo `build.rs` that runs `curl | sh` at compile time.

**Impact:** Code execution at install or build time on every developer machine and every CI/CD runner that installs the package. The process runs before any application code review occurs.

**Code review signs:**
- `package.json` files (in any dependency, not just direct) containing `preinstall`, `postinstall`, or `install` script entries.
- `build.rs` files in Cargo dependencies that invoke network commands (`curl`, `wget`, `fetch`) or execute shell commands.
- Package manager not configured with `ignore-scripts=true` (npm/pnpm equivalent).
- CI/CD pipelines that run `npm install` or `cargo build` without sandboxing or network restriction.

---

## 6. Lockfile Manipulation

**Description:** The lockfile (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Cargo.lock`) is supposed to guarantee reproducible installs by pinning exact versions and checksums. An attacker with write access to the repository modifies the lockfile to change the `resolved` URL of a package to point to a malicious tarball, while keeping the package name and version number unchanged. `npm install` fetches the malicious tarball without warning.

**Vector:** Pull requests that modify the lockfile without a corresponding change to the manifest (`package.json`), or CI/CD pipelines that run `npm install` (not `npm ci`) which regenerates the lockfile from scratch.

**Impact:** Silently installs malicious code disguised as a legitimate package. Passes `npm audit` (the advisory database checks package name and version, not the tarball URL).

**Real incident:** `num2words` v0.5.15 (PyPI, 2025): a patch-level update delivered via Dependabot automated merging, demonstrating that automatic updaters are a viable vector.

**Code review signs:**
- Lockfile changes in a PR with no corresponding manifest (`package.json`, `Cargo.toml`) changes.
- `resolved` URLs in `package-lock.json` pointing to non-canonical registries (not `registry.npmjs.org`).
- CI/CD using `npm install` instead of `npm ci` (the latter fails if lockfile is out of sync).
- No `lockfile-lint` or equivalent tool in the CI pipeline validating that all resolved URLs use HTTPS and point to authorized registries.

---

## 7. CI/CD Pipeline Injection

**Description:** GitHub Actions and similar CI/CD platforms execute third-party actions. If an action is referenced by a mutable tag (e.g., `@v4`, `@main`) rather than a pinned commit SHA, an attacker who compromises the action's repository can push malicious code that runs in every downstream workflow.

**Real incident:** CVE-2025-30066 — `tj-actions/changed-files` (GitHub Actions, 2025, CVSS: Critical): A popular CI/CD action was compromised. The malicious version logged all CI/CD environment secrets — including `GITHUB_TOKEN`, `AWS_ACCESS_KEY_ID`, and `AWS_SECRET_ACCESS_KEY` — into **public** workflow logs visible to anyone. Affected any repository using this action.

**Code review signs:**
- GitHub Actions workflows using `uses: some-action/tool@v4` or `@main` (mutable references).
- No pinning to exact commit SHA (e.g., `uses: actions/checkout@abc123def456...`).
- CI/CD secrets not scoped to minimal permissions (use `permissions:` block at job level).
- Dependabot/Renovate configured to auto-merge CI/CD action updates without human review.

---

## 8. Namespace Hijacking, Star Jacking, and Provenance Bypass

**Namespace hijacking:** After a maintainer deletes their account, an attacker re-registers the namespace. Pipelines pinned only to `username/package` (without a version or commit hash) silently install the attacker-controlled version. Real incident: HuggingFace namespace reuse (2025); ML pipelines loading models by `username/model` without a pinned revision hash were vulnerable.

**Star jacking:** An attacker acquires a GitHub account with many stars, renames its repository to match the malicious package, and uses the inherited star count to simulate popularity. LLMs evaluating package quality are also misled by inflated star counts.

**Provenance attestation bypass:** Cryptographic provenance (Sigstore / npm Trusted Publishing) proves *where* a package was built, not that the source code is safe. An attacker who compromises a maintainer's GitHub account or CI/CD pipeline can publish a malicious release with a valid provenance attestation. Demonstrated by Shai-Hulud: after stealing npm tokens, the worm could generate valid attestations for malicious releases of victim packages.

**Code review signs:**
- Dependencies referenced by name/namespace only — no pinned version, commit SHA, or content hash.
- ML model references without a pinned revision hash.
- Provenance badge treated as a sufficient trust signal without review of recent commit history.
- No process for auditing new releases of critical dependencies before deployment.

---

## Universal Code Review Checklist

- [ ] All direct dependencies pinned to exact versions (no floating ranges).
- [ ] Lockfile committed and installed via a strict command (`npm ci`, not `npm install`).
- [ ] `preinstall` / `postinstall` scripts absent or reviewed in all direct dependencies.
- [ ] `build.rs` / build-time scripts in dependencies reviewed for network or shell calls.
- [ ] CI/CD actions pinned to commit SHAs, not mutable version tags.
- [ ] Install commands use `--ignore-scripts` flag in CI pipelines.
- [ ] CI/CD secrets scoped with minimal permissions; confirmed absent from public logs.
- [ ] Dependabot / Renovate auto-merge disabled or limited to pre-vetted updates.
- [ ] New packages verified: creation date, download volume, maintainer count.
- [ ] Package names cross-checked against official documentation — never only an LLM suggestion.
- [ ] Internal package names registered on public registries as placeholders (dependency confusion defense).
- [ ] Lockfile URLs validated against an allowlist of authorized registries.
