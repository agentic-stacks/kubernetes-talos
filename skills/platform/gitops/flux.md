# Platform — GitOps: Flux

Flux is a set of continuous delivery controllers for Kubernetes that reconcile cluster state from Git, OCI registries, and Helm repositories. This guide covers Flux installation, CRD usage, multi-tenancy, monitoring, and Talos-specific bootstrap patterns.

---

## 1. Bootstrap Flux with GitHub

The `flux bootstrap github` command installs Flux controllers in the cluster and creates (or updates) a Git repository with the Flux system manifests. It is idempotent — running it again updates Flux in place.

### Prerequisites

```bash
# Install the Flux CLI
# macOS
brew install fluxcd/tap/flux

# Linux
curl -s https://fluxcd.io/install.sh | sudo bash

# Verify prerequisites — checks Kubernetes version and cluster connectivity
flux check --pre
```

### Bootstrap Command

```bash
# Export your GitHub personal access token
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Bootstrap Flux into the cluster
flux bootstrap github \
  --owner=your-org \
  --repository=cluster-config \
  --branch=main \
  --path=clusters/production \
  --personal \
  --token-auth

# Explanation of flags:
#   --owner         GitHub user or organization that owns the repository
#   --repository    Repository name (created if it does not exist)
#   --branch        Branch to use as the source of truth
#   --path          Path within the repo for this cluster's manifests
#   --personal      Use a personal access token (vs. GitHub App)
#   --token-auth    Use HTTPS token auth (vs. SSH deploy key)
```

For SSH-based authentication (deploy key):

```bash
flux bootstrap github \
  --owner=your-org \
  --repository=cluster-config \
  --branch=main \
  --path=clusters/production \
  --ssh-key-algorithm=ed25519
```

For a private repository with a specific team:

```bash
flux bootstrap github \
  --owner=your-org \
  --repository=cluster-config \
  --branch=main \
  --path=clusters/production \
  --team=platform-team \
  --private=true \
  --token-auth
```

### What Bootstrap Creates

After bootstrap, the repository at `clusters/production/flux-system/` contains:

```
clusters/production/flux-system/
  gotk-components.yaml   # All Flux CRDs and controller Deployments
  gotk-sync.yaml         # GitRepository and Kustomization for self-management
  kustomization.yaml     # Kustomize entrypoint
```

The cluster now has these controllers running in `flux-system`:

| Controller | Purpose |
|---|---|
| source-controller | Fetches artifacts from Git, Helm, and OCI repositories |
| kustomize-controller | Applies Kustomize overlays and plain YAML manifests |
| helm-controller | Manages Helm releases via HelmRelease CRDs |
| notification-controller | Sends and receives event notifications |

---

## 2. Source CRDs

### GitRepository

Tracks a Git repository and makes its contents available to other Flux controllers.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: app-repo
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/your-org/app-repo.git
  ref:
    branch: main
  secretRef:
    name: app-repo-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: app-repo-auth
  namespace: flux-system
type: Opaque
stringData:
  username: git
  password: ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

For SSH authentication:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: app-repo
  namespace: flux-system
spec:
  interval: 5m
  url: ssh://git@github.com/your-org/app-repo.git
  ref:
    branch: main
  secretRef:
    name: app-repo-ssh
---
apiVersion: v1
kind: Secret
metadata:
  name: app-repo-ssh
  namespace: flux-system
type: Opaque
stringData:
  identity: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
  known_hosts: |
    github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl
```

Tag-based references for pinned versions:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: app-repo
  namespace: flux-system
spec:
  interval: 1h
  url: https://github.com/your-org/app-repo.git
  ref:
    tag: v1.2.3
```

### HelmRepository

Tracks a Helm chart repository (HTTP/HTTPS index-based).

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami
```

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: jetstack
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.jetstack.io
```

For authenticated Helm repositories:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: private-charts
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.example.com
  secretRef:
    name: private-charts-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: private-charts-auth
  namespace: flux-system
type: Opaque
stringData:
  username: chart-reader
  password: s3cret-t0ken
```

### OCIRepository

Tracks OCI artifacts (container registry-hosted Helm charts or plain manifests).

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 5m
  url: oci://ghcr.io/stefanprodan/manifests/podinfo
  ref:
    tag: latest
```

For AWS ECR:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: app-manifests
  namespace: flux-system
spec:
  interval: 5m
  url: oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/manifests/app
  ref:
    tag: v2.0.0
  provider: aws
```

---

## 3. Kustomization CRD

The Flux Kustomization CRD (not to be confused with the Kustomize `kustomization.yaml` file) tells kustomize-controller to apply a set of manifests from a source.

### Basic Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure-controllers
  namespace: flux-system
spec:
  interval: 1h
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/controllers
  prune: true
  wait: true
```

### Dependency Ordering

Kustomizations can declare dependencies to enforce ordering. This ensures cert-manager is fully healthy before cluster-issuers are applied:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure-controllers
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/controllers
  prune: true
  wait: true
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure-configs
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/configs
  prune: true
  dependsOn:
    - name: infrastructure-controllers
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: platform
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./platform
  prune: true
  dependsOn:
    - name: infrastructure-configs
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/production
  prune: true
  dependsOn:
    - name: platform
```

### Health Checks

Kustomizations can define custom health checks to wait for specific resources:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cert-manager
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/controllers/cert-manager
  prune: true
  wait: true
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: cert-manager
      namespace: cert-manager
    - apiVersion: apps/v1
      kind: Deployment
      name: cert-manager-webhook
      namespace: cert-manager
```

### Variable Substitution

Kustomizations support post-build variable substitution from ConfigMaps and Secrets:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/production
  prune: true
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-settings
      - kind: Secret
        name: cluster-secrets
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-settings
  namespace: flux-system
data:
  CLUSTER_NAME: production
  CLUSTER_DOMAIN: prod.example.com
  INGRESS_CLASS: nginx
```

Then in manifests, use `${CLUSTER_DOMAIN}` and it will be replaced at apply time.

### HelmRelease

HelmRelease manages Helm chart installations with automatic upgrades, rollbacks, and drift detection.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  interval: 1h
  chart:
    spec:
      chart: cert-manager
      version: "1.17.x"
      sourceRef:
        kind: HelmRepository
        name: jetstack
        namespace: flux-system
      interval: 1h
  install:
    crds: CreateReplace
    remediation:
      retries: 3
  upgrade:
    crds: CreateReplace
    remediation:
      retries: 3
      remediateLastFailure: true
  values:
    installCRDs: false
    replicaCount: 2
    podDnsPolicy: None
    podDnsConfig:
      nameservers:
        - 1.1.1.1
        - 8.8.8.8
```

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  interval: 1h
  chart:
    spec:
      chart: ingress-nginx
      version: "4.12.x"
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
  values:
    controller:
      replicaCount: 2
      service:
        type: LoadBalancer
      metrics:
        enabled: true
        serviceMonitor:
          enabled: true
```

---

## Next Steps

- [Flux operations](flux-operations.md) — multi-tenancy, SOPS secret encryption, monitoring, Talos bootstrap, CLI commands, and image automation
