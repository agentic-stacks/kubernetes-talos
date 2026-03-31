# Platform — GitOps

GitOps for Talos Linux clusters. This skill covers the GitOps methodology, tool selection between Flux and ArgoCD, repository structure patterns, and integration with Talos's declarative machine configuration model.

---

## 1. Why GitOps on Talos

Talos Linux is already a declarative, API-driven operating system. Every aspect of the OS and cluster is defined in machine configuration YAML and applied via the Talos API. GitOps extends this model to the application and platform layers:

- **Everything in Git.** Cluster state, platform components, and application deployments are all version-controlled. Git becomes the single source of truth, not the cluster.
- **Declarative all the way down.** Talos machine configs are declarative. Kubernetes manifests are declarative. GitOps controllers reconcile the desired state from Git against the live cluster state continuously.
- **Audit trail by default.** Every change to the cluster is a Git commit. Who changed what, when, and why is captured in the commit history. Rollback is `git revert`.
- **Drift detection and correction.** Manual `kubectl apply` changes are detected and reverted. The cluster always converges to the state described in Git.
- **Reproducibility.** A cluster can be rebuilt from scratch by pointing a GitOps controller at the repository. Combined with Talos machine configs in Git, the entire stack — OS, Kubernetes, platform, and apps — is reproducible.
- **Multi-cluster consistency.** The same repository structure can target multiple clusters with environment-specific overlays, ensuring consistency across dev, staging, and production.

### The Talos + GitOps Philosophy

Talos removes SSH, shell access, and package managers. The only way to manage Talos is through its API. This aligns perfectly with GitOps: humans do not touch the cluster directly. They commit to Git, and automation reconciles the desired state.

The full stack becomes:

```
Git Repository (source of truth)
  |
  +--> Talos Machine Configs --> talosctl apply / Talos API
  |
  +--> Kubernetes Manifests  --> Flux or ArgoCD --> Kubernetes API
```

---

## 2. Flux vs ArgoCD

Both are CNCF projects. Both implement the GitOps model. They differ in architecture, operational philosophy, and feature set.

### Comparison Table

| Aspect | Flux | ArgoCD |
|---|---|---|
| **Architecture** | Set of independent controllers (source-controller, kustomize-controller, helm-controller, notification-controller) running as Deployments | Monolithic application server with API server, repo server, application controller, Redis cache, and optional Dex for SSO |
| **UI** | No built-in UI. Weave GitOps (OSS or Enterprise) adds a dashboard. CLI-first workflow. | Built-in web UI with application topology visualization, diff views, sync status, and log streaming |
| **CRD Model** | GitRepository, HelmRepository, OCIRepository, Kustomization, HelmRelease — composable primitives | Application, ApplicationSet, AppProject — higher-level abstractions |
| **Multi-tenancy** | Native. Each Kustomization can impersonate a ServiceAccount scoped to a namespace. Tenants get their own GitRepository and Kustomization resources. | AppProject-based. Projects define allowed source repos, destination clusters/namespaces, and resource whitelists. RBAC policies control user access. |
| **Helm Support** | HelmRelease CRD with drift detection, dependency ordering, and automatic rollback on failed upgrades | Helm charts rendered at sync time. Values files from Git. No native Helm rollback — uses Git revert. |
| **Kustomize Support** | Native. Kustomization CRD applies kustomize overlays with dependency ordering and health checks. | Native. Kustomize overlays applied during rendering. |
| **Secret Management** | SOPS and Age decryption built into kustomize-controller. Mozilla SOPS integration is first-class. | External Secrets Operator or Sealed Secrets as separate tools. SOPS via a plugin. |
| **Notifications** | Built-in notification-controller with Provider CRDs for Slack, Teams, Discord, GitHub, GitLab, PagerDuty, and webhooks | Built-in notifications via argocd-notifications with triggers and templates for Slack, email, webhooks, and more |
| **Cluster API** | Manages remote clusters via Kustomization `kubeConfig` references | First-class multi-cluster via cluster secrets registered in ArgoCD |
| **Resource Footprint** | Lighter. ~200 MB RAM baseline for all controllers. | Heavier. ~500 MB–1 GB RAM baseline. Redis and repo-server add overhead. |
| **Complexity** | Lower operational complexity. Git-native workflow. Everything is a CRD in the cluster. | Higher operational complexity. Separate state (Redis), separate auth (Dex/OIDC), UI server. |
| **Learning Curve** | Steeper initial learning curve (many CRDs, composition model). Simpler once understood. | Gentler initial learning curve (UI-driven). More complex RBAC and project model at scale. |
| **Garbage Collection** | Prune option on Kustomization — deletes resources removed from Git | Auto-prune via sync policy — deletes resources removed from Git |
| **Image Automation** | Built-in image-reflector-controller and image-automation-controller scan registries and commit image tag updates back to Git | No built-in image updater. Argo CD Image Updater is a separate project. |

### Recommendations by Use Case

**Choose Flux when:**
- You want a lightweight, composable GitOps engine
- CLI-first workflow is preferred
- SOPS/Age secret encryption is a requirement
- You are running on resource-constrained nodes (edge, single-node, Raspberry Pi)
- You want image automation that commits back to Git
- Multi-tenancy via ServiceAccount impersonation fits your model

**Choose ArgoCD when:**
- A visual UI for application topology and sync status is important
- Your team includes operators who prefer GUI-based workflows
- You need SSO integration (OIDC, SAML, LDAP) for the management interface
- ApplicationSet generators (Git, cluster, matrix, merge) simplify large-scale rollouts
- You manage many clusters from a single control plane

**For Talos specifically:** Flux is the more natural fit because it aligns with Talos's CLI-first, API-driven philosophy and has a smaller resource footprint. However, ArgoCD works perfectly well and is preferred if UI-based observability is a priority.

---

## 3. Repository Structure Patterns

### Monorepo vs Multi-Repo

**Monorepo** — a single repository contains infrastructure configs, platform components, and application manifests.

```
kubernetes-cluster/
  clusters/
    production/
      flux-system/         # or argocd-system/
      infrastructure.yaml  # Kustomization pointing to infrastructure/
      platform.yaml        # Kustomization pointing to platform/
      apps.yaml            # Kustomization pointing to apps/
    staging/
      flux-system/
      infrastructure.yaml
      platform.yaml
      apps.yaml
  infrastructure/
    controllers/
      cert-manager/
      ingress-nginx/
      external-dns/
    configs/
      cluster-issuer.yaml
      dns-zone.yaml
  platform/
    monitoring/
      kube-prometheus-stack/
    logging/
      loki/
    gitops/
      flux/ or argocd/
  apps/
    production/
      app-a/
      app-b/
    staging/
      app-a/
      app-b/
```

Advantages: single PR touches all layers, easy cross-cutting refactors, simple CI.
Disadvantages: blast radius of access control, noisy commit history, harder to isolate app team permissions.

**Multi-Repo** — separate repositories for infrastructure, platform, and each application.

```
repo: cluster-infrastructure    --> cert-manager, ingress, external-dns
repo: cluster-platform          --> monitoring, logging, gitops tooling
repo: app-a                     --> application manifests and Helm chart
repo: app-b                     --> application manifests and Helm chart
repo: cluster-config            --> top-level Kustomizations/Applications pointing to the above
```

Advantages: fine-grained access control per repo, clean separation of concerns, app teams own their repos.
Disadvantages: more repositories to manage, cross-repo dependency coordination is harder.

**Recommendation:** Start with a monorepo. Split into multi-repo only when team boundaries or access control requirements demand it.

### Infrastructure / Platform / Apps Layering

Use three layers with explicit dependency ordering:

1. **Infrastructure** — cluster-level controllers that other components depend on. Installed first.
   - cert-manager (TLS certificates)
   - ingress-nginx or Cilium Gateway API
   - external-dns
   - external-secrets-operator

2. **Platform** — observability, security, and developer tooling. Depends on infrastructure.
   - kube-prometheus-stack (monitoring)
   - Loki + Promtail (logging)
   - Velero (backup)
   - GitOps controller itself (Flux or ArgoCD)

3. **Apps** — business applications. Depends on platform and infrastructure.
   - Application Helm releases or kustomize overlays
   - Per-environment configuration

In Flux this ordering is enforced via `dependsOn` on Kustomization resources. In ArgoCD it is enforced via sync waves or explicit sync ordering.

### Environment Promotion

**Branch-per-environment (not recommended):** Leads to long-lived branches, merge conflicts, and configuration drift between environments.

**Directory-per-environment (recommended):** A single branch (`main`) with directories for each environment. Promotion is a PR that copies or updates manifests from one directory to another.

```
clusters/
  dev/
    apps/
      kustomization.yaml    # patches: replicas=1, image tag from dev
  staging/
    apps/
      kustomization.yaml    # patches: replicas=2, image tag promoted from dev
  production/
    apps/
      kustomization.yaml    # patches: replicas=3, image tag promoted from staging
apps/
  base/
    deployment.yaml
    service.yaml
    kustomization.yaml
```

Promotion workflow:
1. CI builds a new image, tags it `v1.2.3`, pushes to registry.
2. Image automation (Flux) or a CI job updates the image tag in `clusters/dev/`.
3. After validation in dev, a PR promotes the tag to `clusters/staging/`.
4. After validation in staging, a PR promotes the tag to `clusters/production/`.

---

## 4. Integration with Talos

### Bootstrapping GitOps via cluster.inlineManifests

Talos supports `cluster.inlineManifests` in the machine configuration. These manifests are applied by Talos directly during cluster bootstrap, before any GitOps controller is running. This creates a self-managing cluster: Talos applies the GitOps controller, and from that point forward the GitOps controller manages everything — including itself.

#### Bootstrap Flux via Talos inlineManifests

Add the Flux namespace and GitRepository/Kustomization to the control-plane machine config:

```yaml
# controlplane.yaml (partial — cluster section)
cluster:
  inlineManifests:
    - name: flux-namespace
      contents: |
        apiVersion: v1
        kind: Namespace
        metadata:
          name: flux-system
    - name: flux-install
      contents: |
        # Output of: flux install --export
        # This contains all Flux CRDs and controllers.
        # Generate with:
        #   flux install --export > flux-install.yaml
        # Then paste the contents here, or reference via a URL.
        #
        # For a real deployment, generate the full output and inline it.
        # The output is ~3000 lines of YAML containing:
        #   - CustomResourceDefinitions for GitRepository, Kustomization, etc.
        #   - Deployments for source-controller, kustomize-controller,
        #     helm-controller, notification-controller
        #   - ServiceAccounts, ClusterRoles, ClusterRoleBindings
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
    - name: flux-git-secret
      contents: |
        apiVersion: v1
        kind: Secret
        metadata:
          name: flux-system
          namespace: flux-system
        type: Opaque
        data:
          # base64-encoded GitHub personal access token or deploy key
          username: Z2l0
          password: Z2hwX3h4eHh4eHh4eHh4eHh4eHh4eHh4eHh4eHh4eA==
```

#### Bootstrap ArgoCD via Talos inlineManifests

```yaml
# controlplane.yaml (partial — cluster section)
cluster:
  inlineManifests:
    - name: argocd-namespace
      contents: |
        apiVersion: v1
        kind: Namespace
        metadata:
          name: argocd
    - name: argocd-install
      contents: |
        # Output of: kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.14.3/manifests/install.yaml
        # Download, review, and inline the full manifest.
        # For a real deployment, download the manifest and paste it here.
    - name: argocd-root-app
      contents: |
        apiVersion: argoproj.io/v1alpha1
        kind: Application
        metadata:
          name: root
          namespace: argocd
        spec:
          project: default
          source:
            repoURL: https://github.com/your-org/cluster-config.git
            targetRevision: main
            path: clusters/production
          destination:
            server: https://kubernetes.default.svc
          syncPolicy:
            automated:
              prune: true
              selfHeal: true
```

### Generating Flux Install Manifests for Inline Use

```bash
# Generate the full Flux install manifests suitable for inlining
flux install --export > flux-install.yaml

# Check the size — typically ~3000 lines
wc -l flux-install.yaml

# To inline into the Talos machine config, paste the contents
# under cluster.inlineManifests[].contents
```

### Important Considerations

- **Secret management:** The Git credentials secret shown above contains a base64-encoded token. For production, use SOPS-encrypted secrets or External Secrets Operator. The inline secret bootstraps initial access; the GitOps controller can then manage its own secret rotation.
- **Self-management:** Once the GitOps controller is running, it can manage its own upgrades. Include the Flux/ArgoCD manifests in the Git repository so the controller reconciles its own version.
- **Machine config updates:** Changes to `cluster.inlineManifests` require a machine config apply via `talosctl`. After the GitOps controller is running, prefer managing resources through Git rather than through inline manifests.
- **Order of operations:** Talos applies inline manifests in order. Place the namespace first, then CRDs and controllers, then GitRepository/Application resources that reference those CRDs.

---

## Next Steps

- [Flux detailed guide](flux.md) — bootstrap, CRDs, multi-tenancy, monitoring
- [ArgoCD detailed guide](argocd.md) — installation, Application patterns, RBAC, SSO, CLI usage
