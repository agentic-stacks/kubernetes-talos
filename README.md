# kubernetes-talos

Complete operational knowledge for deploying, managing, and operating production Kubernetes clusters using [Talos Linux](https://www.talos.dev). An [agentic stack](https://github.com/agentic-stacks/agentic-stacks) that turns AI agents into Talos experts.

## What This Is

This repository is a structured knowledge base — not code. It teaches AI agents (Claude, Gemini, GPT, etc.) how to operate Kubernetes clusters on Talos Linux across the full lifecycle: from initial infrastructure provisioning through production operations, upgrades, and disaster recovery.

When an agent reads the `CLAUDE.md` entry point, it gains:
- Deep understanding of Talos's immutable, API-driven architecture
- Step-by-step procedures with exact `talosctl` and `kubectl` commands
- Decision frameworks for choosing CNI, storage, ingress, and other components
- Safety guardrails that prevent destructive operations without approval
- Symptom-based troubleshooting decision trees

## Coverage

| Area | What's Covered |
|---|---|
| **Deployment targets** | Bare metal, Proxmox, VMware, libvirt, AWS, GCP, Azure, Hetzner, DigitalOcean |
| **CNI** | Cilium (7 install methods), Flannel, Calico, Multus |
| **Storage** | Rook-Ceph, Longhorn, OpenEBS, Local Path, NFS, Cloud CSI (EBS/PD/Azure Disk) |
| **Platform** | Flux, ArgoCD, NGINX/Traefik/Cilium Gateway API, cert-manager, Prometheus, Loki, OpenTelemetry, Kyverno/OPA, Sealed Secrets/ESO/Vault, Cilium Mesh/Istio/Linkerd |
| **Operations** | Health checks, node scaling, Talos + K8s upgrades, etcd backup/DR, CA rotation |
| **Topologies** | Single-node dev, 3 CP + N workers, 5 CP + N workers, multi-site |
| **Talos versions** | 1.8.x, 1.9.x with version-specific known issues |

## Quick Start

### For AI Agent Users

Pull this stack into your project:

```bash
agentic-stacks init ./my-cluster --name my-cluster --namespace my-org --from kubernetes-talos
```

Or clone directly:

```bash
git clone https://github.com/agentic-stacks/kubernetes-talos.git .stacks/kubernetes-talos
```

Then point your agent to `CLAUDE.md` (or `.stacks/kubernetes-talos/CLAUDE.md` if using the stacks workflow). The agent will use the routing table to navigate to the right skill for any task.

### For Humans

Browse the skills directly:

- **New to Talos?** Start with [`skills/foundation/concepts`](skills/foundation/concepts/README.md)
- **Building a cluster?** Follow the [new cluster workflow](#workflows)
- **Choosing components?** Check the [decision guides](skills/reference/decision-guides/README.md)
- **Something broken?** Jump to [troubleshooting](skills/diagnose/troubleshooting/README.md)

## Skills

### Foundation
| Skill | Description |
|---|---|
| [`foundation/concepts`](skills/foundation/concepts/README.md) | Talos architecture, immutable OS model, API-driven operations |
| [`foundation/machine-config`](skills/foundation/machine-config/README.md) | Config generation, patching, secrets management, system extensions |
| [`foundation/infrastructure`](skills/foundation/infrastructure/README.md) | Platform-specific provisioning guides for 9 platforms |

### Deploy
| Skill | Description |
|---|---|
| [`deploy/bootstrap`](skills/deploy/bootstrap/README.md) | Cluster creation, `talosctl bootstrap`, kubeconfig retrieval |
| [`deploy/networking`](skills/deploy/networking/README.md) | CNI selection, comparison, and installation (Cilium, Flannel, Calico, Multus) |
| [`deploy/storage`](skills/deploy/storage/README.md) | CSI selection, comparison, and installation (6 options) |

### Platform
| Skill | Description |
|---|---|
| [`platform/gitops`](skills/platform/gitops/README.md) | Flux and ArgoCD bootstrap, repo structure patterns |
| [`platform/ingress`](skills/platform/ingress/README.md) | NGINX, Traefik, Cilium Gateway API, cert-manager |
| [`platform/observability`](skills/platform/observability/README.md) | Prometheus, Loki, OpenTelemetry, Talos-native metrics |
| [`platform/security`](skills/platform/security/README.md) | Pod security, secrets management, RBAC, network policy |
| [`platform/service-mesh`](skills/platform/service-mesh/README.md) | Cilium mesh, Istio, Linkerd |

### Operations
| Skill | Description |
|---|---|
| [`operations/health-check`](skills/operations/health-check/README.md) | Cluster validation procedures and health report format |
| [`operations/scaling`](skills/operations/scaling/README.md) | Adding and removing nodes, topology changes |
| [`operations/upgrades`](skills/operations/upgrades/README.md) | Talos OS, Kubernetes, and component rolling upgrades |
| [`operations/backup-restore`](skills/operations/backup-restore/README.md) | etcd backup, Velero, disaster recovery procedures |
| [`operations/certificate-mgmt`](skills/operations/certificate-mgmt/README.md) | Talos PKI, CA rotation, expiry monitoring |

### Diagnose
| Skill | Description |
|---|---|
| [`diagnose/troubleshooting`](skills/diagnose/troubleshooting/README.md) | Symptom-based decision trees for 8 common scenarios |

### Reference
| Skill | Description |
|---|---|
| [`reference/known-issues`](skills/reference/known-issues/README.md) | Version-specific bugs and workarounds |
| [`reference/compatibility`](skills/reference/compatibility/README.md) | Talos/K8s/CNI/CSI compatibility matrices |
| [`reference/decision-guides`](skills/reference/decision-guides/README.md) | Trade-off analyses for CNI, CSI, topology, HA, GitOps |

## Workflows

### New Cluster

```
foundation/concepts → foundation/machine-config → foundation/infrastructure
→ deploy/bootstrap → deploy/networking → deploy/storage
→ platform/* (as needed) → operations/health-check
```

### Existing Cluster

Jump directly to the relevant `operations/`, `diagnose/`, or `platform/` skill.

## Required Tools

| Tool | Purpose |
|---|---|
| `talosctl` | Talos API client (version must match target Talos version) |
| `kubectl` | Kubernetes CLI |
| `helm` | Helm package manager |
| `flux` | Flux CLI (optional, for GitOps with Flux) |
| `argocd` | ArgoCD CLI (optional, for GitOps with ArgoCD) |

## Project Structure

When using this stack, your operator project should look like:

```
my-cluster/
├── CLAUDE.md                # Points to .stacks/kubernetes-talos/
├── stacks.lock
├── .stacks/
│   └── kubernetes-talos/    # This stack
├── controlplane.yaml        # Generated machine config
├── controlplane.yaml.orig   # Stock config for diffing
├── worker.yaml
├── worker.yaml.orig
├── secrets.yaml             # Cluster secrets (keep secure)
├── patches/                 # Per-role and per-node patches
├── talosconfig              # Talos client config
├── kubeconfig               # K8s client config
├── manifests/               # Platform component manifests
└── scripts/                 # Operational scripts
```

## Contributing

This stack follows the [agentic-stacks](https://github.com/agentic-stacks/agentic-stacks) format. Each skill is a directory under `skills/` with a `README.md` entry point and optional sub-files for specific topics.

To add or update content:
1. Follow the existing writing style (imperative headings, exact commands, decision trees)
2. Verify commands against the [official Talos docs](https://docs.siderolabs.com)
3. Add version-specific notes where behavior differs between Talos releases
4. Update `stack.yaml` if adding new skills

## License

MIT
