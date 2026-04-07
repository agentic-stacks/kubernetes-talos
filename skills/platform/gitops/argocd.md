# Platform — GitOps: ArgoCD

ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. It provides a web UI, CLI, and API for managing application deployments from Git repositories. This guide covers installation, Application patterns, RBAC, SSO, sync policies, and CLI usage.

---

## 1. Installation via Helm

### Add the ArgoCD Helm Repository

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### Install with Custom Values

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 7.8.0 \
  --values argocd-values.yaml
```

### Helm Values File

```yaml
# argocd-values.yaml
global:
  domain: argocd.example.com

configs:
  params:
    # Run in insecure mode if TLS is terminated at the ingress
    server.insecure: true
  cm:
    # Enable status badge for repositories
    statusbadge.enabled: true
    # Kustomize build options
    kustomize.buildOptions: --enable-helm
    # Resource tracking method — annotation is more reliable than label
    application.resourceTrackingMethod: annotation
  rbac:
    # Default RBAC policy — deny all, grant explicitly
    policy.default: role:readonly
    policy.csv: |
      p, role:platform-admin, applications, *, */*, allow
      p, role:platform-admin, clusters, get, *, allow
      p, role:platform-admin, repositories, *, *, allow
      p, role:platform-admin, projects, *, *, allow
      p, role:platform-admin, logs, get, *, allow
      p, role:platform-admin, exec, create, */*, allow
      p, role:developer, applications, get, */*, allow
      p, role:developer, applications, sync, */*, allow
      p, role:developer, logs, get, */*, allow
      g, platform-admins, role:platform-admin
      g, developers, role:developer

server:
  replicas: 2
  ingress:
    enabled: true
    ingressClassName: nginx
    hostname: argocd.example.com
    tls: true
    extraTls:
      - hosts:
          - argocd.example.com
        secretName: argocd-tls

controller:
  replicas: 1
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      namespace: argocd

repoServer:
  replicas: 2
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true

applicationSet:
  replicas: 2
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true

redis-ha:
  enabled: true

notifications:
  enabled: true
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
```

### Minimal Installation (Non-HA, Single Node)

For development or resource-constrained clusters:

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 7.8.0 \
  --set server.replicas=1 \
  --set controller.replicas=1 \
  --set repoServer.replicas=1 \
  --set applicationSet.replicas=1 \
  --set redis-ha.enabled=false \
  --set configs.params."server\.insecure"=true
```

### Retrieve Initial Admin Password

```bash
# The initial admin password is stored in a Kubernetes secret
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Login with the CLI
argocd login argocd.example.com --username admin --password <password>

# Change the admin password immediately
argocd account update-password \
  --account admin \
  --current-password <initial-password> \
  --new-password <new-password>

# Delete the initial admin secret after changing the password
kubectl -n argocd delete secret argocd-initial-admin-secret
```

---

## 2. Application CRD

The Application CRD is the core resource in ArgoCD. It defines a source (Git repo, Helm chart, or OCI artifact) and a destination (cluster and namespace).

### Basic Application from Git

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/cluster-config.git
    targetRevision: main
    path: apps/nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Application from Helm Chart

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: v1.17.1
    helm:
      releaseName: cert-manager
      valuesObject:
        installCRDs: true
        replicaCount: 2
        podDnsPolicy: None
        podDnsConfig:
          nameservers:
            - 1.1.1.1
            - 8.8.8.8
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Application with Kustomize

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/app.git
    targetRevision: main
    path: deploy/overlays/production
    kustomize:
      images:
        - ghcr.io/your-org/app:v1.2.3
  destination:
    server: https://kubernetes.default.svc
    namespace: app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Application with Multiple Sources

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 69.8.0
      helm:
        releaseName: kube-prometheus-stack
        valueFiles:
          - $values/monitoring/values-production.yaml
    - repoURL: https://github.com/your-org/cluster-config.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

---

## 3. App-of-Apps Pattern

The app-of-apps pattern uses a single root Application that points to a directory containing other Application manifests. This creates a hierarchy where ArgoCD manages all applications from a single entry point.

### Root Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/cluster-config.git
    targetRevision: main
    path: clusters/production
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Directory Structure

```
clusters/production/
  infrastructure/
    cert-manager.yaml      # Application for cert-manager
    ingress-nginx.yaml     # Application for ingress-nginx
    external-dns.yaml      # Application for external-dns
  platform/
    monitoring.yaml        # Application for kube-prometheus-stack
    logging.yaml           # Application for Loki
  apps/
    app-a.yaml             # Application for app-a
    app-b.yaml             # Application for app-b
  kustomization.yaml       # Kustomize entrypoint including all subdirectories
```

### Kustomization Entrypoint

```yaml
# clusters/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - infrastructure/cert-manager.yaml
  - infrastructure/ingress-nginx.yaml
  - infrastructure/external-dns.yaml
  - platform/monitoring.yaml
  - platform/logging.yaml
  - apps/app-a.yaml
  - apps/app-b.yaml
```

### Sync Waves for Ordering

ArgoCD uses sync waves to control the order of resource creation. Lower wave numbers sync first.

```yaml
# infrastructure/cert-manager.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: v1.17.1
    helm:
      valuesObject:
        installCRDs: true
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```yaml
# infrastructure/cluster-issuer.yaml — depends on cert-manager
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cluster-issuer
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/cluster-config.git
    targetRevision: main
    path: infrastructure/configs/cert-manager
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```yaml
# platform/monitoring.yaml — depends on infrastructure
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "10"
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 69.8.0
    helm:
      valuesObject:
        grafana:
          ingress:
            enabled: true
            ingressClassName: nginx
            hosts:
              - grafana.example.com
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

```yaml
# apps/app-a.yaml — depends on platform
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "20"
spec:
  project: apps
  source:
    repoURL: https://github.com/your-org/app-a.git
    targetRevision: main
    path: deploy/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: app-a
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 4. ApplicationSet

ApplicationSet generates Application resources from templates combined with generators. It replaces manual creation of repetitive Application manifests.

### Git Directory Generator

Creates one Application per directory in a Git repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions:
    - missingkey=error
  generators:
    - git:
        repoURL: https://github.com/your-org/cluster-config.git
        revision: main
        directories:
          - path: apps/*
  template:
    metadata:
      name: "{{ .path.basename }}"
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/cluster-config.git
        targetRevision: main
        path: "{{ .path.path }}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{ .path.basename }}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Cluster Generator

Creates one Application per registered cluster (useful for multi-cluster deployments):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: platform-monitoring
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  template:
    metadata:
      name: "monitoring-{{ .name }}"
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://prometheus-community.github.io/helm-charts
        chart: kube-prometheus-stack
        targetRevision: 69.8.0
      destination:
        server: "{{ .server }}"
        namespace: monitoring
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Matrix Generator

Combines two generators (e.g., clusters x apps):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-apps
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - matrix:
        generators:
          - clusters:
              selector:
                matchLabels:
                  environment: production
          - git:
              repoURL: https://github.com/your-org/cluster-config.git
              revision: main
              directories:
                - path: apps/*
  template:
    metadata:
      name: "{{ .name }}-{{ .path.basename }}"
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/cluster-config.git
        targetRevision: main
        path: "{{ .path.path }}"
      destination:
        server: "{{ .server }}"
        namespace: "{{ .path.basename }}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### List Generator

Explicit list of parameters:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: environments
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - list:
        elements:
          - env: dev
            replicas: "1"
            branch: develop
          - env: staging
            replicas: "2"
            branch: main
          - env: production
            replicas: "3"
            branch: main
  template:
    metadata:
      name: "app-{{ .env }}"
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/app.git
        targetRevision: "{{ .branch }}"
        path: deploy/overlays/{{ .env }}
      destination:
        server: https://kubernetes.default.svc
        namespace: "app-{{ .env }}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## Next Steps

- [ArgoCD operations](argocd-operations.md) — RBAC, SSO, sync policies, notifications, CLI commands, Talos bootstrap, and monitoring
