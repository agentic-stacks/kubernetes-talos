# Decision Guide — CNI Selection

## Context

The Container Network Interface (CNI) provides pod-to-pod networking, service load balancing, and network policy enforcement. In Talos Linux, this is a **bootstrap-time decision** that cannot be changed without rebuilding the cluster.

**When to decide:** Before running `talosctl gen config` or `talosctl apply-config` for the first time.

**Can it be changed later?** No. Switching CNI requires a full cluster rebuild. Choose carefully.

**Talos-specific constraint:** If using any CNI other than the built-in Flannel, you must set `cluster.network.cni.name: none` and `cluster.proxy.disabled: true` (if using kube-proxy replacement) in the machine config before bootstrap.

---

## Options

### Cilium

The most feature-rich option. Uses eBPF for high-performance dataplane processing, replaces kube-proxy entirely, and provides L3-L7 network policies, transparent encryption, service mesh capabilities, and Gateway API support.

**Strengths:**
- eBPF-based dataplane with kube-proxy replacement — eliminates iptables overhead
- L3-L7 network policies including DNS-aware policies
- Native Gateway API support (ingress without a separate controller)
- Transparent encryption via WireGuard or IPsec
- Hubble observability (network flow logs, service map)
- Excellent Talos integration — actively tested and documented by Sidero Labs
- Bandwidth Manager for pod traffic shaping
- Cluster Mesh for multi-cluster networking

**Weaknesses:**
- Higher resource consumption than Flannel (~200-300MB RAM per node for the agent)
- More complex configuration surface
- Requires specific Talos settings (cgroup mounts, security contexts)
- Debugging requires understanding eBPF concepts

**Talos-required settings:**

```yaml
# Machine config
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true

# Cilium Helm values (minimum required for Talos)
cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup
kubeProxyReplacement: true
ipam:
  mode: kubernetes
securityContext:
  capabilities:
    ciliumAgent: "{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}"
    cleanCiliumState: "{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}"
bpf:
  masquerade: true
forwardKubeDNSToHost: false
```

### Flannel

The simplest option. Built into Talos as the default CNI. VXLAN overlay network with no network policy support.

**Strengths:**
- Zero installation — works out of the box with default Talos config
- Minimal resource usage (~30-50MB RAM per node)
- Simple to understand and debug
- No additional configuration required
- Stable and battle-tested

**Weaknesses:**
- No NetworkPolicy support — pods can communicate freely
- No kube-proxy replacement — relies on iptables-based kube-proxy
- No encryption, no observability, no advanced features
- VXLAN only — no native routing option
- Cannot be extended with additional features

**Talos config:** Default — no changes needed:

```yaml
cluster:
  network:
    cni:
      name: flannel   # This is the default, can be omitted
  proxy:
    disabled: false    # kube-proxy required
```

### Calico

A mature CNI with strong network policy support. Offers both iptables and eBPF dataplanes. The eBPF mode provides kube-proxy replacement.

**Strengths:**
- L3-L4 network policies (industry standard)
- eBPF dataplane available (kube-proxy replacement)
- BGP peering for native routing (useful in bare-metal environments)
- WireGuard encryption
- Large community and extensive documentation
- Operator-based lifecycle management (Tigera operator)

**Weaknesses:**
- No L7 (application-layer) network policies in open-source version
- No Gateway API support
- eBPF mode less mature than Cilium's
- iptables mode does NOT work on Talos (kernel module loading restriction)
- Must use eBPF mode or VXLAN mode on Talos
- No built-in observability equivalent to Hubble

**Talos config for Calico eBPF mode:**

```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

---

## Comparison Table

| Feature | Cilium | Flannel | Calico |
|---------|--------|---------|--------|
| **Installation complexity** | Medium | None (built-in) | Medium |
| **Resource usage per node** | ~200-300MB RAM | ~30-50MB RAM | ~150-250MB RAM |
| **Dataplane** | eBPF | VXLAN (kernel) | eBPF or VXLAN |
| **Kube-proxy replacement** | Yes | No | Yes (eBPF mode) |
| **Network policies** | L3-L7 | None | L3-L4 |
| **DNS-aware policies** | Yes | No | No |
| **Encryption** | WireGuard, IPsec | No | WireGuard |
| **Gateway API** | Yes (native) | No | No |
| **Service mesh** | Yes (sidecar-free) | No | No |
| **Observability** | Hubble (flows, map) | None | Basic flow logs |
| **Multi-cluster** | Cluster Mesh | No | Federation |
| **BGP support** | Yes | No | Yes (native) |
| **Bandwidth management** | Yes | No | No |
| **IPv6 / dual-stack** | Yes | Yes | Yes |
| **Talos compatibility** | Excellent | Built-in | Good (eBPF mode) |
| **iptables mode on Talos** | N/A | N/A | NOT supported |
| **Debugging ease** | Medium (eBPF tooling) | Easy | Medium |
| **Community size** | Large, growing fast | Large | Large |
| **Commercial support** | Isovalent (Cisco) | N/A | Tigera |

---

## Recommendation by Use Case

| Use Case | Recommendation | Reasoning |
|----------|---------------|-----------|
| **Production cluster** | Cilium | Network policies, observability, kube-proxy replacement, encryption |
| **Homelab / learning** | Flannel (simple) or Cilium (learn eBPF) | Flannel if you want zero friction; Cilium if you want production-like experience |
| **Edge / resource-constrained** | Flannel | Minimal resource overhead; Cilium if you need policies despite resource cost |
| **Multi-tenant** | Cilium | L7 policies, DNS-aware policies, namespace isolation |
| **Bare-metal with BGP** | Calico or Cilium | Both support BGP; Calico has more mature BGP implementation |
| **Gateway API required** | Cilium | Only CNI with native Gateway API support |
| **Compliance / audit** | Cilium | Hubble provides network flow visibility for audit trails |
| **Air-gapped / minimal** | Flannel | No external dependencies, no container images to pull beyond Talos defaults |

---

## Migration Path

**Flannel to Cilium / Calico:** Requires cluster rebuild. There is no in-place migration path because:
1. `cluster.network.cni.name` cannot be changed on a running cluster
2. `cluster.proxy.disabled` cannot be toggled without rebuild
3. Pod CIDR allocations and routing tables are incompatible

**Cilium to Calico (or reverse):** Also requires cluster rebuild for the same reasons.

**Migration procedure:**
1. Document all workloads, configurations, and persistent data
2. Back up etcd: `talosctl etcd snapshot -n <cp-node> /tmp/etcd-backup.snapshot`
3. Back up persistent volumes (Velero or manual)
4. Generate new cluster config with desired CNI settings
5. Build new cluster, install new CNI
6. Restore workloads (GitOps makes this trivial — point Flux/ArgoCD at the new cluster)
7. Restore persistent data from backups

**Key takeaway:** Get the CNI decision right the first time. If unsure, Cilium is the safest choice for any cluster that might grow beyond a simple homelab.
