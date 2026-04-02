# Operations: Upgrades

Strategy and pre-flight procedures for upgrading a Talos-managed Kubernetes cluster. There are three independent upgrade dimensions — each has its own sub-document:

| Upgrade Type | What Changes | Document |
|-------------|-------------|----------|
| **Talos OS** | Node operating system, containerd, kernel, etcd binary | [talos-upgrade.md](talos-upgrade.md) |
| **Kubernetes** | API server, kubelet, scheduler, controller manager, kube-proxy | [kubernetes-upgrade.md](kubernetes-upgrade.md) |
| **Components** | CNI, CSI, ingress controllers, observability, Helm charts | [component-upgrade.md](component-upgrade.md) |

These upgrades are **independent**. You can upgrade Talos OS without upgrading Kubernetes, and vice versa. Component upgrades (Helm charts) are entirely separate from both.

---

## Upgrade Strategy Overview

**Order of operations for a full stack upgrade:**

```
1. Upgrade Talos OS on control-plane nodes (one at a time)
2. Upgrade Talos OS on worker nodes (one at a time)
3. Upgrade Kubernetes version (talosctl upgrade-k8s)
4. Upgrade platform components (CNI, CSI, etc.)
```

Always upgrade the control plane before workers. Always complete Talos OS upgrades before Kubernetes upgrades — the new Talos version may ship an updated kubelet that the new Kubernetes version requires.

**Never skip steps.** Never upgrade workers before control-plane nodes. Never upgrade Kubernetes before Talos if both need upgrading.

---

## Pre-Flight Checklist

Complete every item before starting any upgrade. Do not skip items.

### 1. Back Up etcd

```bash
talosctl etcd snapshot db.snapshot --nodes 192.168.1.10
# Store the snapshot somewhere off-cluster (S3, local machine, etc.)
ls -la db.snapshot
```

If the upgrade goes catastrophically wrong and etcd is lost, this snapshot is your recovery path. See [backup-restore](../backup-restore/README.md).

### 2. Verify Current Health

```bash
talosctl health --nodes 192.168.1.10
talosctl etcd status --nodes 192.168.1.10
talosctl etcd alarm list --nodes 192.168.1.10
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

**Never upgrade a cluster that is not fully healthy.** Fix all issues first.

### 3. Read Release Notes

- **Talos:** https://www.talos.dev/latest/introduction/releases/
- **Kubernetes:** https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/
- Look for: breaking changes, deprecated features, required actions, minimum version requirements

### 4. Check Known Issues

```bash
# Check the Talos GitHub issues for your target version
# https://github.com/siderolabs/talos/issues?q=label%3A%22release-note%22

# Check Kubernetes known issues
# https://github.com/kubernetes/kubernetes/issues
```

### 5. Verify Version Compatibility

Talos ships a specific range of supported Kubernetes versions. Confirm your target Kubernetes version is supported by your target (or current) Talos version.

```bash
# Current versions
talosctl version --nodes 192.168.1.10
kubectl version

# Check compatibility matrix at:
# https://www.talos.dev/latest/introduction/support-matrix/
```

**Key rule:** Talos N.M supports Kubernetes versions from the Talos release's embedded kubelet version through several minor versions. The support matrix is the authoritative source.

### 6. Verify PodDisruptionBudgets

PDBs protect workloads during node drains. Upgrades trigger rolling reboots, which drain nodes.

```bash
# List all PDBs — check that none will block draining
kubectl get pdb --all-namespaces

# Look for PDBs where ALLOWED DISRUPTIONS is 0 — these will block upgrades
kubectl get pdb --all-namespaces -o wide | awk '$6 == 0'
```

If a PDB has `ALLOWED DISRUPTIONS: 0`, either:
- Scale up the workload so disruptions are allowed
- Temporarily relax the PDB (risky for production)
- Accept that the upgrade will pause at that node until the PDB allows draining

### 7. Verify Sufficient Capacity

During rolling upgrades, one node at a time goes offline. The remaining nodes must handle the displaced workloads.

```bash
kubectl top nodes
kubectl describe nodes | grep -A 3 "Allocated resources"
```

### 8. (Optional) Test in Staging

If you have a staging cluster, run the full upgrade there first. Verify:
- All workloads survive the rolling reboot cycle
- No PDBs blocked the process
- Application health checks pass after upgrade
- Any custom admission webhooks or operators work with the new versions
