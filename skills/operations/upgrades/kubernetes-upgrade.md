# Kubernetes Upgrade

Upgrading the Kubernetes version running on a Talos cluster. This is separate from upgrading Talos OS — the two are independent operations.

---

## How It Works

Talos manages the Kubernetes upgrade through a single command:

```bash
talosctl upgrade-k8s --to 1.32.1 --nodes 192.168.1.10
```

This command handles the entire rolling upgrade automatically:
1. Upgrades static pod manifests for API server, controller manager, scheduler on each CP node
2. Upgrades kubelet on every node (control-plane and workers)
3. Upgrades kube-proxy (if in use)
4. Rolls components one at a time to maintain availability

You point `--nodes` at **one** control-plane node. Talos coordinates the upgrade across the entire cluster from there.

---

## Version Compatibility

### Talos-to-Kubernetes Matrix

Each Talos minor version supports a specific range of Kubernetes versions. Check the official support matrix before upgrading:

```
https://www.talos.dev/latest/introduction/support-matrix/
```

Example (verify against current matrix):
```
Talos v1.9.x  -> Kubernetes 1.30.x through 1.32.x
Talos v1.8.x  -> Kubernetes 1.29.x through 1.31.x
```

**Rule:** If you need a Kubernetes version that your current Talos does not support, upgrade Talos first.

### Kubernetes Version Skew

Kubernetes itself supports upgrading one minor version at a time:

```
v1.30.x -> v1.31.x    (supported)
v1.31.x -> v1.32.x    (supported)
v1.30.x -> v1.32.x    (NOT supported — step through v1.31.x)
```

Patch version jumps within the same minor are always fine:

```
v1.31.2 -> v1.31.7    (supported)
```

---

## Upgrade Procedure

### Pre-Flight

```bash
# 1. Verify current versions
talosctl version --nodes 192.168.1.10
kubectl version

# 2. Verify cluster health
talosctl health --nodes 192.168.1.10
kubectl get nodes -o wide
kubectl get pods -n kube-system

# 3. Take etcd snapshot
talosctl etcd snapshot db.snapshot --nodes 192.168.1.10

# 4. Check that PDBs allow disruptions
kubectl get pdb --all-namespaces
```

### Dry Run

```bash
# Preview what will happen without making changes
talosctl upgrade-k8s --to 1.32.1 --nodes 192.168.1.10 --dry-run
```

The dry run shows which components will be upgraded and their version transitions.

### Execute the Upgrade

```bash
talosctl upgrade-k8s --to 1.32.1 --nodes 192.168.1.10
```

**What happens during the upgrade:**

1. **API server** manifests are updated on each CP node, one at a time. The API server restarts with the new version. Other CP nodes continue serving requests.
2. **Controller manager** and **scheduler** are updated similarly.
3. **kubelet** is updated on each node. The node briefly goes NotReady as kubelet restarts, then returns to Ready.
4. **kube-proxy** DaemonSet (if present) is updated.

The entire process typically takes 5-15 minutes for a 3 CP + 3 worker cluster, depending on pod restart times and PDB constraints.

### Monitor the Upgrade

```bash
# In another terminal, watch node versions change
kubectl get nodes -w

# Watch kube-system pods rolling
kubectl get pods -n kube-system -w

# After completion, verify all components
kubectl version
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

### Post-Upgrade Verification

```bash
# Verify Kubernetes version on all nodes
kubectl get nodes -o wide
# All nodes should show the new kubelet version

# Verify API server version
kubectl version
# Server Version should show the target version

# Verify kube-system pods are healthy
kubectl get pods -n kube-system

# Full health check
talosctl health --nodes 192.168.1.10

# Check for any pods in bad states
kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded
```

---

## Kubernetes Upgrade vs. Talos Upgrade

| Aspect | `talosctl upgrade` (Talos OS) | `talosctl upgrade-k8s` (Kubernetes) |
|--------|-------------------------------|--------------------------------------|
| What changes | Kernel, initramfs, containerd, etcd binary, system services | API server, kubelet, scheduler, controller manager, kube-proxy |
| Reboots nodes | Yes | No (kubelet restarts, but no node reboot) |
| Affects etcd data | No (data preserved across reboot) | No |
| Command target | Each node individually, one at a time | One CP node; Talos coordinates the rest |
| Rollback | `talosctl rollback` (A/B partitions) | Run `talosctl upgrade-k8s --to <old-version>` |
| Duration per node | 2-5 minutes (reboot cycle) | 30-60 seconds (process restart) |

**Typical full-stack upgrade sequence:**
```
1. talosctl upgrade --nodes <cp-1> --image ...   (repeat for each CP)
2. talosctl upgrade --nodes <w-1> --image ...     (repeat for each worker)
3. talosctl upgrade-k8s --to 1.32.1 --nodes <cp-1>
```

---

## Rollback

If the Kubernetes upgrade causes issues, downgrade by running the same command with the previous version:

```bash
talosctl upgrade-k8s --to 1.31.5 --nodes 192.168.1.10
```

**Caveats:**
- Kubernetes API objects that were created using features introduced in the new version may not work after downgrade
- CRDs with new API versions may need manual cleanup
- Always check the Kubernetes changelog for deprecation and removal notices before rolling back

---

## Troubleshooting

### Upgrade hangs on a node

```bash
# Check what is happening on the stuck node
talosctl services --nodes <stuck-node>
talosctl logs kubelet --nodes <stuck-node>

# Check if a PDB is blocking pod eviction
kubectl get pdb --all-namespaces -o wide | awk '$6 == 0'
```

### API server not responding after upgrade

```bash
# Check API server logs on each CP node
talosctl logs kube-apiserver --nodes 192.168.1.10
talosctl logs kube-apiserver --nodes 192.168.1.11
talosctl logs kube-apiserver --nodes 192.168.1.12

# Verify etcd is healthy — API server depends on etcd
talosctl etcd status --nodes 192.168.1.10
```

### kubelet fails to start on new version

```bash
talosctl logs kubelet --nodes <node-ip>

# Common cause: version skew — kubelet version must be within one minor of API server
# Ensure you upgraded CP nodes before workers
```
