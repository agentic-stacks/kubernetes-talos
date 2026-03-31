# Reference — Compatibility Matrix

Agent reference for version compatibility between Talos Linux, Kubernetes, CNIs, CSIs, and platform components. Consult this before recommending versions for new clusters or planning upgrades.

---

## 1. Talos Linux to Kubernetes Version Matrix

Talos supports a specific range of Kubernetes versions. Using an unsupported combination will result in bootstrap failures or runtime instability.

| Talos Version | Kubernetes 1.29 | Kubernetes 1.30 | Kubernetes 1.31 | Kubernetes 1.32 | Default K8s | EOL |
|---------------|:---:|:---:|:---:|:---:|---|---|
| 1.8.0-1.8.x  | Yes | Yes | Yes | No  | 1.30.x | Approaching EOL |
| 1.9.0-1.9.x  | No  | Yes | Yes | Yes | 1.31.x | Current |

**Key rules:**
- Always use the **default Kubernetes version** for a Talos release unless the user has a specific reason to diverge.
- Talos N.M typically supports Kubernetes versions from the previous release plus one newer version.
- Kubernetes version is set in machine config at `cluster.kubernetesVersion` or via `talosctl gen config --kubernetes-version`.
- Upgrading Kubernetes independently of Talos: `talosctl upgrade-k8s --to <version> -n <cp-node>`

---

## 2. CNI Compatibility

### Cilium

| Cilium Version | Talos 1.8.x | Talos 1.9.x | Kubernetes Range | Notes |
|---|:---:|:---:|---|---|
| 1.15.x | Yes | Yes | 1.27-1.31 | LTS release |
| 1.16.x | Yes | Yes | 1.29-1.32 | Current stable |
| 1.17.x | No  | Yes | 1.30-1.32 | Latest, requires Talos 1.9+ kernel features |

**Talos-specific Cilium settings (always required):**

```yaml
cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup
```

**Recommended Cilium settings for Talos:**

```yaml
kubeProxyReplacement: true
bpf:
  masquerade: true
forwardKubeDNSToHost: false
ipam:
  mode: kubernetes
securityContext:
  capabilities:
    ciliumAgent: "{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}"
    cleanCiliumState: "{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}"
```

### Flannel

| Flannel Version | Talos 1.8.x | Talos 1.9.x | Notes |
|---|:---:|:---:|---|
| Built-in (bundled) | Yes | Yes | Default CNI, no installation required |
| 0.25.x (self-managed) | Yes | Yes | For Helm-managed Flannel |
| 0.26.x (self-managed) | Yes | Yes | Latest |

Flannel is bundled with Talos and requires no installation when using the default `cluster.network.cni.name: flannel` setting.

### Calico

| Calico Version | Talos 1.8.x | Talos 1.9.x | Kubernetes Range | Notes |
|---|:---:|:---:|---|---|
| 3.27.x | Yes | Yes | 1.27-1.31 | Use eBPF mode for kube-proxy replacement |
| 3.28.x | Yes | Yes | 1.29-1.32 | Current stable |
| 3.29.x | No  | Yes | 1.30-1.32 | Latest |

**Talos-specific Calico notes:**
- Must use eBPF dataplane mode — iptables mode requires kernel module loading which Talos does not support from workloads.
- Install via Tigera operator, not manifest-based install.

---

## 3. CSI Compatibility

### Storage Providers

| CSI Provider | Talos 1.8.x | Talos 1.9.x | Required Extensions | Notes |
|---|:---:|:---:|---|---|
| Rook-Ceph 1.14.x-1.15.x | Yes | Yes | None (runs in pods) | Production distributed storage |
| Longhorn 1.6.x-1.7.x | Yes | Yes | `iscsi-tools`, `util-linux-tools` | Requires Talos system extensions |
| OpenEBS (Mayastor) 2.6.x+ | Yes | Yes | `iscsi-tools` | NVMe-oF based, high performance |
| Local Path Provisioner 0.0.28+ | Yes | Yes | None | Single-node only, no replication |
| NFS Subdir Provisioner 4.0.x | Yes | Yes | `nfs-client` (extension) | External NFS server required |
| democratic-csi (iSCSI/NFS) | Yes | Yes | `iscsi-tools` | TrueNAS, FreeNAS, ZFS backends |

**Installing Talos system extensions for CSI:**

Extensions are added to the machine config at install time or via an upgrade:

```yaml
machine:
  install:
    extensions:
      - image: ghcr.io/siderolabs/iscsi-tools:v0.1.6
      - image: ghcr.io/siderolabs/util-linux-tools:v2.40.2
```

After modifying extensions, apply the config and upgrade the node for extensions to take effect:

```bash
talosctl upgrade -n <node> --image ghcr.io/siderolabs/installer:v1.9.x
```

### Cloud CSI Drivers

| Cloud Provider | CSI Driver | Talos Support | Notes |
|---|---|:---:|---|
| AWS | aws-ebs-csi-driver | Yes | EBS volumes |
| GCP | gcp-pd-csi-driver | Yes | Persistent Disks |
| Azure | azuredisk-csi-driver | Yes | Azure Managed Disks |
| Hetzner | hcloud-csi-driver | Yes | Hetzner Cloud Volumes |
| Proxmox | N/A | N/A | No native CSI; use Ceph, Longhorn, or NFS |

---

## 4. Platform Component Compatibility

### cert-manager

| cert-manager Version | Talos 1.8.x | Talos 1.9.x | Kubernetes Range | Notes |
|---|:---:|:---:|---|---|
| 1.14.x | Yes | Yes | 1.27-1.31 | |
| 1.15.x | Yes | Yes | 1.28-1.32 | Current stable |
| 1.16.x | No  | Yes | 1.30-1.32 | Latest |

No Talos-specific configuration required. Install via Helm.

### Ingress Controllers

| Ingress Controller | Versions | Talos Support | Notes |
|---|---|:---:|---|
| ingress-nginx | 1.10.x-1.12.x | Yes | No special config needed |
| Cilium Ingress | (bundled with Cilium) | Yes | Requires Cilium CNI |
| Traefik | 3.0.x-3.2.x | Yes | No special config needed |
| Envoy Gateway | 1.1.x-1.2.x | Yes | Gateway API implementation |

### Monitoring Stack

| Component | Versions | Talos Support | Notes |
|---|---|:---:|---|
| kube-prometheus-stack (Helm) | 60.x-68.x | Yes | Prometheus + Grafana + Alertmanager |
| Victoria Metrics | 0.25.x-0.28.x (operator) | Yes | Lower resource usage alternative |
| Grafana Alloy (agent) | 1.0.x-1.4.x | Yes | Replaces Grafana Agent |

### GitOps

| Tool | Versions | Talos Support | Notes |
|---|---|:---:|---|
| Flux | 2.3.x-2.4.x | Yes | Recommended for Talos clusters |
| ArgoCD | 2.11.x-2.13.x | Yes | Full support |

---

## 5. Version Selection Guidelines

1. **New clusters:** Use the latest Talos release with its default Kubernetes version. Pick the latest stable Cilium that supports that Kubernetes version.
2. **Upgrades:** Upgrade Talos first, then Kubernetes, then CNI. Never skip Talos minor versions (1.8 -> 1.9 is fine, 1.7 -> 1.9 is not).
3. **Extensions:** Check the [Talos extensions repository](https://github.com/siderolabs/extensions) for the exact extension image tags matching your Talos version. Extension versions are tied to Talos releases.
4. **When in doubt:** Consult `talosctl gen config --help` for the default Kubernetes version, and the Talos release notes for tested component versions.
