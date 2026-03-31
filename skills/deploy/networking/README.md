# Deploy — Networking: CNI Selection Guide

## Talos CNI Model

Talos Linux ships with **Flannel as the default CNI**. To use a custom CNI, you must set `cluster.network.cni.name: none` in your machine configuration **before bootstrapping** the cluster. This cannot be easily changed after the fact — CNI choice is a bootstrap-time decision.

```yaml
cluster:
  network:
    cni:
      name: none
```

## Decision Matrix

| Feature | Cilium | Flannel | Calico |
|---|---|---|---|
| eBPF dataplane | Yes | No | Yes (eBPF mode) |
| Kube-proxy replacement | Yes | No | Yes (eBPF mode) |
| NetworkPolicy | Yes (L3-L7) | No | Yes (L3-L4) |
| Encryption (WireGuard) | Yes | No | Yes |
| Gateway API | Yes | No | No |
| Complexity | Medium | Low | Medium |
| Talos integration | Excellent | Built-in default | Good |

## Kube-Proxy Replacement

Cilium and Calico (eBPF mode) can replace kube-proxy entirely using eBPF. To enable this, set the following **before bootstrapping**:

```yaml
cluster:
  proxy:
    disabled: true
```

Both Cilium and Calico eBPF mode then take over service routing at the kernel level, eliminating the kube-proxy DaemonSet.

## Talos-Specific Gotchas

- **No kernel module loading**: Talos does not allow Kubernetes workloads to load kernel modules. CNIs that rely on kernel module loading (e.g., iptables modules in certain Calico modes) will not work. Use eBPF-based dataplanes.
- **cgroup settings**: Talos manages cgroup mounts. Set `cgroup.autoMount.enabled=false` and `cgroup.hostRoot=/sys/fs/cgroup` when installing Cilium.
- **Bootstrap hang**: After setting `cni.name: none`, the cluster will appear to hang at phase 18/19 during bootstrap until the CNI is installed. This is expected. A reboot occurs automatically after a 10-minute timeout if CNI is not installed in time.
- **Decide before bootstrap**: CNI selection is effectively permanent. Switching after bootstrap requires cluster rebuild.

## CNI Guides

- [Cilium](./cilium.md) — recommended for production; eBPF, kube-proxy replacement, Gateway API
- [Flannel](./flannel.md) — default; simplest option, no NetworkPolicy
- [Calico](./calico.md) — eBPF mode, L3-L4 NetworkPolicy, operator-based
- [Multus](./multus.md) — multi-CNI for pods requiring multiple network interfaces
