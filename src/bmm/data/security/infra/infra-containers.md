# Infra Containers — Security Patterns
> Tags: docker, kubernetes, container, isolation
> Load when: project uses Docker, Kubernetes, or any container runtime

## CVEs Critiques

| CVE | CVSS | Service | Impact | Fix |
|-----|------|---------|--------|-----|
| CVE-2024-21626 | — | runc | Leaky Vessels : fd leak via `WORKDIR /proc/self/fd/7/` → filesystem hôte. Metasploit dispo | runc ≥ 1.1.12 |
| CVE-2025-31133 | — | runc | Race `/dev/null` → symlink → `core_pattern` → code exec hôte | runc v1.2.8+ |
| CVE-2025-52881 | — | runc/crun/youki | Contourne tous les LSMs, redirige vers procfs → full host takeover | runc v1.2.8+ |
| CVE-2025-52565 | — | runc | `/dev/console` bind-mount race → écriture procfs | runc v1.2.8+ |
| CVE-2025-9074 | 9.3 | Docker Desktop | API `192.168.65.7:2375` sans auth → 2 HTTP POST = takeover hôte | Docker Desktop 4.44.3 |
| CVE-2025-47290 | 7.6 | containerd | TOCTOU image unpacking → écriture hors container | containerd 2.1.1 |
| CVE-2025-1974 | 9.8 | ingress-nginx | IngressNightmare : upload lib via client body → injection `ssl_engine` → RCE + tous les secrets cluster | v1.12.1+ / v1.11.5+ |
| CVE-2025-1097/1098/24514 | 8.8 | ingress-nginx | Config injection via annotations non sanitisées | v1.12.1+ |

## Failles Spécifiques

### Container privilégié

```yaml
# VULN
privileged: true

# FIX
read_only: true
security_opt: [no-new-privileges:true]
cap_drop: [ALL]
cap_add: [NET_BIND_SERVICE]  # uniquement si port < 1024 requis
```

### Secret en ARG/layer Dockerfile (persiste après rm)

```dockerfile
# VULN — token visible dans layers de l'image
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc \
    && pip install pkg && rm .npmrc

# FIX — BuildKit secret mount (jamais écrit en layer)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc pip install pkg
# Usage : docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp .
```

### Docker socket exposé (= root hôte)

```bash
# Exploit immédiat
docker run -it -v /:/host/ ubuntu chroot /host/ bash

# FIX : proxy filtrant en lecture seule
# tecnativa/docker-socket-proxy avec POST: 0 (aucune écriture)
```

### Image non pinnée → supply chain

```dockerfile
# VULN
FROM ubuntu:22.04

# FIX — digest immuable
FROM ubuntu:22.04@sha256:703218c0465075f4425e58fac086e09e1de5c340b12976ab9eb8ad26615c3715
```

## Kubernetes — Pod Hardening

```yaml
# RBAC — VULN : cluster-admin wildcard
roleRef:
  kind: ClusterRole
  name: cluster-admin

# RBAC — FIX : namespace-scoped, verbs précis
kind: Role
metadata:
  namespace: app-namespace
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
```

```yaml
# Pod spec sécurisé (tous ces champs obligatoires)
spec:
  automountServiceAccountToken: false
  hostPID: false
  hostNetwork: false
  hostIPC: false
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    resources:
      limits:
        cpu: "500m"
        memory: "256Mi"
```

```yaml
# Enforce cluster-wide Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

## Secrets K8s — base64 ≠ chiffrement

`kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d` → immédiat.

```yaml
# FIX 1 — etcd encryption at rest
kind: EncryptionConfiguration
resources:
- resources: [secrets]
  providers:
  - aescbc: {keys: [{name: key1, secret: <base64-32-bytes>}]}
  - identity: {}

# FIX 2 — External Secrets Operator (prod)
kind: ExternalSecret
spec:
  refreshInterval: "15m"
  secretStoreRef: {name: vault-backend, kind: SecretStore}
  data:
  - secretKey: password
    remoteRef: {key: production/database, property: password}
```

## Kubelet exposé (port 10250)

287 000 sur Shodan (~100 anonymes). FIX kubelet config : `authentication.anonymous.enabled: false`, `readOnlyPort: 0`, `authorization.mode: Webhook`.

## Hardening Checklist

- [ ] runc ≥ 1.2.8 + containerd ≥ 2.1.1 sur tous les noeuds
- [ ] Aucun container `privileged: true` en production
- [ ] Docker socket jamais monté dans un container (ou proxy filtrant POST:0)
- [ ] Images pinnées par digest SHA256, pas par tag
- [ ] Secrets via BuildKit `--mount=type=secret`, jamais en ARG/ENV
- [ ] RBAC : aucun ClusterRoleBinding vers `cluster-admin` pour les workloads
- [ ] `automountServiceAccountToken: false` par défaut sur tous les pods
- [ ] Namespace enforce `pod-security.kubernetes.io/enforce: restricted`
- [ ] etcd encryption at rest + External Secrets Operator en prod
- [ ] Kubelet : `anonymous.enabled: false`, `readOnlyPort: 0`, `authorization.mode: Webhook`
- [ ] ingress-nginx ≥ v1.12.1 (patch IngressNightmare CVE-2025-1974)
- [ ] Network Policies : deny-all par défaut, whitelist explicite
