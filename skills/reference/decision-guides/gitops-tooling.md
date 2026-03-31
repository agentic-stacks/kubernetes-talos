# Decision Guide — GitOps Tooling

## Context

GitOps tools synchronize Kubernetes cluster state with a Git repository, enabling declarative, auditable, and automated infrastructure management. The two dominant tools in the ecosystem are Flux and ArgoCD.

**When to decide:** After cluster bootstrap, before deploying workloads. GitOps should be the first thing installed after CNI and CSI.

**Can it be changed later?** Yes, but migration requires re-expressing all workload definitions for the new tool's format and re-establishing sync. It is disruptive but not destructive.

**Talos-specific note:** Neither tool requires Talos-specific configuration. Both work identically on Talos as on any Kubernetes distribution. The choice is purely about workflow, team preferences, and operational requirements.

---

## Options

### Flux

A set of Kubernetes controllers that continuously reconcile cluster state with Git repositories, Helm charts, OCI artifacts, and S3 buckets. Flux is a CNCF graduated project.

**Architecture:**
- Multiple controllers, each responsible for a specific source/reconciliation type:
  - `source-controller` — fetches from Git, Helm, OCI, S3
  - `kustomize-controller` — applies Kustomize overlays
  - `helm-controller` — manages HelmRelease resources
  - `notification-controller` — webhooks, alerts, event forwarding
  - `image-reflector-controller` / `image-automation-controller` — image update automation
- All controllers run in the `flux-system` namespace
- Configuration is entirely via Kubernetes CRDs (GitRepository, Kustomization, HelmRelease, etc.)
- CLI tool (`flux`) for bootstrapping and management

**Strengths:**
- Pull-based model — cluster pulls from Git, no CI/CD pipeline needs cluster credentials
- Kustomize-native — first-class support for Kustomize overlays and multi-environment configs
- Helm-native — HelmRelease CRD for declarative Helm chart management with values in Git
- Multi-tenancy via namespaced CRDs — teams manage their own GitRepository/Kustomization resources
- Lightweight — total resource usage ~200-400MB RAM across all controllers
- No UI by default — everything is CLI/CRD-driven (UI available via Weave GitOps add-on)
- OCI artifact support — store Kubernetes manifests in container registries
- Strong secret management integration (SOPS, age, Sealed Secrets)
- Dependency ordering via `dependsOn` fields in Kustomization resources
- Automatic image update detection and Git commit

**Weaknesses:**
- No built-in UI — requires third-party add-on (Weave GitOps) for visualization
- Steeper learning curve for CRD-based configuration (no web form to fill out)
- Debugging requires kubectl and flux CLI — no click-through troubleshooting
- Multi-cluster management requires bootstrapping Flux on each cluster separately
- Rollback is Git-based (revert the commit), not a button click

**Installation:**

```bash
flux bootstrap github \
  --owner=<github-org> \
  --repository=<repo-name> \
  --branch=main \
  --path=clusters/<cluster-name> \
  --personal
```

### ArgoCD

A declarative GitOps continuous delivery tool with a rich web UI, SSO integration, and application-centric management. ArgoCD is a CNCF graduated project.

**Architecture:**
- Monolithic application with several components:
  - `argocd-server` — API server and web UI
  - `argocd-repo-server` — clones Git repos, renders manifests (Helm, Kustomize, plain YAML)
  - `argocd-application-controller` — watches Application CRDs and syncs cluster state
  - `argocd-redis` — caching layer
  - `argocd-dex-server` — SSO/OIDC integration
  - `argocd-applicationset-controller` — generates Applications from templates
- Central `Application` CRD defines what to deploy, from where, and to which cluster
- Web UI for visualization, diff views, sync actions, and rollback
- CLI tool (`argocd`) for automation and scripting

**Strengths:**
- Rich web UI — real-time visualization of application state, resource tree, diffs, logs
- Multi-cluster management from a single ArgoCD instance (hub-spoke model)
- ApplicationSets — template-driven generation of Applications for fleet management
- SSO/RBAC — integrate with OIDC, LDAP, SAML for team access control
- Sync windows — control when syncs can happen (maintenance windows)
- Manual sync approval — require human approval before applying changes
- Rollback via UI — one-click rollback to previous revision
- Extensive plugin ecosystem (Vault, SOPS, custom config management tools)
- Health assessment — custom health checks for CRDs

**Weaknesses:**
- Higher resource usage (~500MB-1GB RAM for all components, more with heavy usage)
- More complex deployment — multiple components, Redis dependency
- UI can become a crutch — teams may bypass Git and sync manually via UI
- Helm support is render-based (Helm template), not native Helm install — no Helm lifecycle hooks
- Multi-tenancy via AppProjects is less flexible than Flux's namespaced model
- repo-server can become a bottleneck with many large repositories
- Secret management requires additional tooling (argocd-vault-plugin, Sealed Secrets)

**Installation:**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Or via Helm:
helm install argocd argo/argo-cd -n argocd --create-namespace
```

---

## Comparison Table

| Feature | Flux | ArgoCD |
|---------|------|--------|
| **CNCF status** | Graduated | Graduated |
| **Architecture** | Multiple focused controllers | Monolithic with multiple components |
| **Web UI** | None built-in (Weave GitOps add-on) | Rich, built-in |
| **CLI** | `flux` | `argocd` |
| **Configuration model** | CRDs in Git (GitRepository, Kustomization, HelmRelease) | Application CRD + web UI |
| **Kustomize support** | Native (kustomize-controller) | Yes (via repo-server) |
| **Helm support** | Native HelmRelease CRD (full Helm lifecycle) | Render-only (helm template, no hooks) |
| **OCI artifacts** | Yes (native) | Yes (since 2.8) |
| **Multi-cluster** | Bootstrap Flux per cluster | Single ArgoCD manages many clusters |
| **Multi-tenancy** | Namespace-scoped CRDs | AppProjects with RBAC |
| **SSO / OIDC** | N/A (no UI to log into) | Built-in (Dex integration) |
| **RBAC** | Kubernetes RBAC | ArgoCD-specific RBAC + K8s RBAC |
| **Image automation** | Built-in (image-reflector + automation controllers) | Not built-in (use ArgoCD Image Updater) |
| **Secret management** | SOPS, age (built-in support) | argocd-vault-plugin (add-on) |
| **Dependency ordering** | `dependsOn` in Kustomization CRDs | Sync waves and hooks |
| **Sync strategy** | Continuous reconciliation (configurable interval) | Auto-sync or manual sync (configurable) |
| **Rollback** | Git revert | Git revert or UI one-click |
| **Notifications** | Built-in (notification-controller) | Built-in (argocd-notifications) |
| **Resource usage** | ~200-400MB RAM total | ~500MB-1GB RAM total |
| **Debugging** | `flux logs`, `kubectl describe`, events | Web UI tree view, diff view, pod logs |
| **Learning curve** | Steeper (CRD-heavy, CLI-driven) | Moderate (UI helps onboarding) |
| **Git repo structure** | Flexible (monorepo or multi-repo) | Flexible (monorepo or multi-repo) |
| **Drift detection** | Continuous (controller loop) | Continuous (controller loop) |
| **Prune / garbage collection** | Built-in (prune: true) | Built-in (auto prune) |
| **Scalability** | Scales well (controllers are independent) | repo-server can bottleneck at scale |

---

## Recommendation by Use Case

| Use Case | Recommendation | Reasoning |
|----------|---------------|-----------|
| **Solo operator / homelab** | Flux | Lightweight, no UI overhead, everything in Git, lower resource cost |
| **Small team (2-5 engineers)** | Either | Flux if team prefers CLI/Git; ArgoCD if team wants visual feedback |
| **Platform team serving multiple app teams** | ArgoCD | UI enables self-service for app teams; RBAC and AppProjects for isolation |
| **Large organization with SSO** | ArgoCD | SSO integration, RBAC, audit logging via UI |
| **Multi-cluster fleet** | ArgoCD (hub-spoke) or Flux (per-cluster) | ArgoCD if centralized management preferred; Flux if each cluster is autonomous |
| **Helm-heavy workflows** | Flux | Native HelmRelease CRD preserves full Helm lifecycle including hooks |
| **Kustomize-heavy workflows** | Either | Both handle Kustomize well |
| **CI/CD integration** | Either | Both support webhooks and reconciliation triggers |
| **Air-gapped / minimal** | Flux | Fewer components, no UI to expose, OCI support for artifact mirroring |
| **Compliance / audit requirements** | ArgoCD | UI provides audit trail, approval workflows, sync windows |
| **Resource-constrained clusters** | Flux | Half the memory footprint of ArgoCD |

---

## Migration Path

### Flux to ArgoCD

1. Install ArgoCD alongside Flux (they can coexist temporarily)
2. For each Flux Kustomization/HelmRelease, create an equivalent ArgoCD Application
3. Disable Flux auto-reconciliation for migrated resources (set `suspend: true` on Flux resources)
4. Verify ArgoCD is managing all resources correctly
5. Uninstall Flux: `flux uninstall`

### ArgoCD to Flux

1. Bootstrap Flux alongside ArgoCD: `flux bootstrap ...`
2. For each ArgoCD Application, create equivalent Flux GitRepository + Kustomization/HelmRelease
3. Delete ArgoCD Applications (set deletion policy to avoid pruning managed resources)
4. Verify Flux is reconciling all resources
5. Uninstall ArgoCD: `kubectl delete -n argocd -f <install-manifest>`

### Key Considerations for Migration

- Both tools use standard Kubernetes manifests, Helm, and Kustomize — the underlying resource definitions do not change
- The migration effort is in re-expressing the "what to sync" and "from where" configuration
- Secret management approach may differ (SOPS for Flux vs Vault plugin for ArgoCD)
- Webhook and notification integrations need to be reconfigured

**Key takeaway:** For Talos clusters managed by a single operator or small team, Flux is the leaner choice. For organizations where a UI, SSO, and multi-cluster management from a single pane are important, ArgoCD is the better fit. Both are production-grade CNCF graduated projects.
