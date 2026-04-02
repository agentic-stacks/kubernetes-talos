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

## 4. Multi-Tenancy

Flux supports multi-tenancy through ServiceAccount impersonation. Each tenant gets a restricted ServiceAccount, and the Kustomization reconciling their resources impersonates that account.

### Tenant Setup

```yaml
# Namespace for the tenant
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
  labels:
    toolkit.fluxcd.io/tenant: team-alpha
---
# ServiceAccount the tenant's Kustomization will impersonate
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-alpha
  namespace: team-alpha
---
# RoleBinding granting the ServiceAccount full access within its namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-reconciler
  namespace: team-alpha
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: team-alpha
    namespace: team-alpha
```

### Tenant GitRepository and Kustomization

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: team-alpha
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/your-org/team-alpha-apps.git
  ref:
    branch: main
  secretRef:
    name: team-alpha-git-auth
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team-alpha-apps
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: team-alpha
  path: ./
  prune: true
  targetNamespace: team-alpha
  serviceAccountName: team-alpha
```

The `serviceAccountName` field causes kustomize-controller to impersonate `team-alpha` in namespace `team-alpha`. The tenant cannot create cluster-scoped resources or resources in other namespaces because the RoleBinding only grants access within `team-alpha`.

### Tenant Cross-Namespace Prevention

To prevent tenants from referencing sources in other namespaces, set this flag on the kustomize-controller deployment:

```yaml
# In the Flux kustomize-controller Deployment args
- --no-cross-namespace-refs=true
```

---

## 5. SOPS Secret Encryption

Flux has built-in support for Mozilla SOPS. Encrypted secrets in Git are decrypted by kustomize-controller at apply time.

### Setup with Age

```bash
# Generate an Age key pair
age-keygen -o age.agekey

# The public key is printed to stdout, e.g.:
# public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p

# Create the SOPS decryption secret in the cluster
cat age.agekey | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin

# Configure the kustomize-controller to use the Age key
# Add to the kustomize-controller Deployment:
#   --decryption-provider=sops
#   --decryption-secret=sops-age
```

### .sops.yaml Configuration

```yaml
# .sops.yaml at the repository root
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

### Encrypting a Secret

```bash
# Create a plain secret
cat <<EOF > secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: default
type: Opaque
stringData:
  username: admin
  password: super-secret-password
EOF

# Encrypt it with SOPS
sops --encrypt --in-place secret.yaml

# The file now has encrypted data fields and SOPS metadata.
# Commit it to Git — kustomize-controller will decrypt it at apply time.
```

### Kustomization with Decryption

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/production
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

---

## 6. Monitoring Flux with Prometheus

Flux controllers expose Prometheus metrics on port 8080 at `/metrics`. Use ServiceMonitor CRDs if you have the Prometheus Operator installed.

### ServiceMonitor for All Flux Controllers

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: flux-system
  namespace: flux-system
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames:
      - flux-system
  selector:
    matchLabels:
      app.kubernetes.io/part-of: flux
  endpoints:
    - port: http-prom
      interval: 30s
      scrapeTimeout: 10s
```

### Key Metrics

| Metric | Description |
|---|---|
| `gotk_reconcile_condition` | Current condition of each Flux resource (Ready, Stalled, etc.) |
| `gotk_reconcile_duration_seconds` | Time taken for each reconciliation |
| `gotk_suspend_status` | Whether a resource is suspended |
| `controller_runtime_reconcile_total` | Total number of reconciliations per controller |
| `controller_runtime_reconcile_errors_total` | Total reconciliation errors per controller |

### Grafana Dashboard

Flux provides an official Grafana dashboard. Import dashboard ID `16714` (Flux Cluster Stats) and `16715` (Flux Control Plane) from grafana.com.

### Alerts

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: flux-alerts
  secretRef:
    name: slack-webhook-url
---
apiVersion: v1
kind: Secret
metadata:
  name: slack-webhook-url
  namespace: flux-system
type: Opaque
stringData:
  address: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: on-call
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: error
  eventSources:
    - kind: Kustomization
      name: "*"
    - kind: HelmRelease
      name: "*"
  summary: "Flux reconciliation failure"
```

For info-level notifications (successful reconciliations):

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: info
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: info
  eventSources:
    - kind: Kustomization
      name: "*"
    - kind: HelmRelease
      name: "*"
```

---

## 7. Bootstrapping Flux via Talos inlineManifests

For a fully self-managing cluster where Talos bootstraps Flux without any external tooling, use `cluster.inlineManifests` in the Talos machine config.

### Step 1: Generate Flux Install Manifests

```bash
# Generate the full set of Flux controller manifests
flux install --export > flux-components.yaml

# Verify the output
kubectl apply --dry-run=client -f flux-components.yaml
```

### Step 2: Add to Talos Machine Config

```yaml
# controlplane.yaml (relevant section only)
cluster:
  inlineManifests:
    - name: flux-namespace
      contents: |
        apiVersion: v1
        kind: Namespace
        metadata:
          name: flux-system
          labels:
            app.kubernetes.io/part-of: flux
            pod-security.kubernetes.io/warn: restricted
            pod-security.kubernetes.io/warn-version: latest
    - name: flux-components
      contents: |
        # Paste the entire contents of flux-components.yaml here.
        # This is typically ~3000 lines containing:
        #   - CRDs: GitRepository, HelmRepository, OCIRepository,
        #           Kustomization, HelmRelease, Provider, Alert, Receiver
        #   - Deployments: source-controller, kustomize-controller,
        #                  helm-controller, notification-controller
        #   - Services, ServiceAccounts, ClusterRoles, ClusterRoleBindings
        #   - NetworkPolicies for inter-controller communication
    - name: flux-sync-secret
      contents: |
        apiVersion: v1
        kind: Secret
        metadata:
          name: flux-system
          namespace: flux-system
        type: Opaque
        data:
          username: Z2l0
          password: <base64-encoded-github-token>
    - name: flux-sync
      contents: |
        apiVersion: source.toolkit.fluxcd.io/v1
        kind: GitRepository
        metadata:
          name: flux-system
          namespace: flux-system
        spec:
          interval: 1m
          url: https://github.com/your-org/cluster-config.git
          ref:
            branch: main
          secretRef:
            name: flux-system
        ---
        apiVersion: kustomize.toolkit.fluxcd.io/v1
        kind: Kustomization
        metadata:
          name: flux-system
          namespace: flux-system
        spec:
          interval: 10m
          path: ./clusters/production
          prune: true
          sourceRef:
            kind: GitRepository
            name: flux-system
          decryption:
            provider: sops
            secretRef:
              name: sops-age
```

### Step 3: Apply Machine Config

```bash
talosctl apply-config --nodes 192.168.1.10 --file controlplane.yaml
```

After the cluster boots, Flux controllers start automatically, connect to the Git repository, and begin reconciling all manifests under `clusters/production/`. From this point forward, Flux manages itself — upgrades to Flux are committed to Git and reconciled like any other resource.

---

## 8. Common CLI Commands

```bash
# Check Flux installation health
flux check

# List all Flux resources and their status
flux get all -A

# List Kustomizations with reconciliation status
flux get kustomizations -A

# List HelmReleases
flux get helmreleases -A

# Trigger an immediate reconciliation
flux reconcile kustomization flux-system --with-source
flux reconcile helmrelease cert-manager -n cert-manager

# Suspend reconciliation (e.g., during maintenance)
flux suspend kustomization apps
flux resume kustomization apps

# View Flux controller logs
flux logs --all-namespaces --level=error

# Export a resource for backup or inspection
flux export kustomization apps > apps-kustomization.yaml

# Uninstall Flux (removes all controllers and CRDs)
flux uninstall --silent
```

---

## 9. Image Automation

Flux can scan container registries for new image tags and automatically commit updates back to Git.

### Install Image Automation Controllers

```bash
flux bootstrap github \
  --owner=your-org \
  --repository=cluster-config \
  --branch=main \
  --path=clusters/production \
  --components-extra=image-reflector-controller,image-automation-controller \
  --token-auth
```

### Image Policy and Update Automation

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: app
  namespace: flux-system
spec:
  image: ghcr.io/your-org/app
  interval: 5m
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: app
  policy:
    semver:
      range: ">=1.0.0"
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@example.com
        name: Flux Image Automation
      messageTemplate: "chore: update image {{range .Changed.Changes}}{{.OldValue}} -> {{.NewValue}} {{end}}"
    push:
      branch: main
  update:
    path: ./clusters/production
    strategy: Setters
```

In the deployment manifest, add a marker comment for the image setter:

```yaml
spec:
  containers:
    - name: app
      image: ghcr.io/your-org/app:1.0.0 # {"$imagepolicy": "flux-system:app"}
```

Flux will update the tag in-place and commit the change back to Git.
