# Talos OS Upgrade

Upgrading the Talos Linux operating system on cluster nodes. This replaces the OS image (kernel, initramfs, containerd, etcd binary, system services) without affecting Kubernetes version.

---

## How Talos Upgrades Work

Talos uses an **A/B partition scheme**. The running OS is on partition A. The upgrade writes the new image to partition B, updates the bootloader to boot from B, and reboots. If the upgrade fails, the node automatically falls back to partition A on the next reboot.

This means:
- Upgrades are atomic — the node either comes up on the new version or rolls back
- The previous version is always available on the other partition
- No partial upgrade states are possible

---

## Supported Upgrade Paths

Talos supports upgrading **one minor version at a time**. You cannot skip minor versions.

```
v1.7.x -> v1.8.x    (supported)
v1.8.x -> v1.9.x    (supported)
v1.7.x -> v1.9.x    (NOT supported — must go through v1.8.x first)
```

Patch version upgrades within the same minor version are always supported:

```
v1.9.0 -> v1.9.2    (supported)
v1.9.1 -> v1.9.5    (supported)
```

---

## Upgrade Image

Talos upgrades require an installer image. There are two sources:

### Standard Image (from ghcr.io)

```
ghcr.io/siderolabs/installer:v1.9.2
```

Use this when your nodes boot from standard Talos media with no custom extensions.

### Image Factory (custom extensions)

If your nodes use system extensions (e.g., QEMU guest agent, iSCSI, ZFS, firmware), you must use an Image Factory-generated installer that includes those extensions. The installer image must match the extensions currently installed on the node.

```bash
# Image Factory URL format:
# https://factory.talos.dev/?arch=amd64&extensions=siderolabs/qemu-guest-agent,siderolabs/iscsi-tools&platform=metal&target=installer&version=v1.9.2

# Example factory-generated installer image:
# factory.talos.dev/installer/<schematic-id>:v1.9.2

# Find your current schematic ID:
talosctl get extensions --nodes 192.168.1.10
```

**Critical:** Using the standard installer on a node that booted with extensions will remove those extensions on upgrade. Always match the installer to the node's extension set.

---

## Upgrade Procedure: One Node at a Time

**Always upgrade control-plane nodes first, then workers.** Never upgrade all nodes simultaneously.

### Upgrading Control-Plane Nodes

```bash
# Pre-flight: verify health and take etcd snapshot
talosctl health --nodes 192.168.1.10
talosctl etcd snapshot db.snapshot --nodes 192.168.1.10

# Upgrade CP node 1
talosctl upgrade --nodes 192.168.1.10 \
  --image ghcr.io/siderolabs/installer:v1.9.2 \
  --wait --debug

# --wait   blocks until the node reboots and comes back healthy
# --debug  streams detailed progress to your terminal

# Verify CP node 1 is back and healthy
talosctl version --nodes 192.168.1.10
talosctl etcd members --nodes 192.168.1.10
talosctl etcd status --nodes 192.168.1.10
kubectl get nodes -o wide

# Upgrade CP node 2
talosctl upgrade --nodes 192.168.1.11 \
  --image ghcr.io/siderolabs/installer:v1.9.2 \
  --wait --debug

# Verify CP node 2
talosctl version --nodes 192.168.1.11
talosctl etcd members --nodes 192.168.1.10
kubectl get nodes -o wide

# Upgrade CP node 3
talosctl upgrade --nodes 192.168.1.12 \
  --image ghcr.io/siderolabs/installer:v1.9.2 \
  --wait --debug

# Verify CP node 3 and full etcd cluster
talosctl version --nodes 192.168.1.12
talosctl etcd members --nodes 192.168.1.10
talosctl etcd status --nodes 192.168.1.10
talosctl health --nodes 192.168.1.10
```

### Upgrading Worker Nodes

```bash
# Upgrade worker 1
talosctl upgrade --nodes 192.168.1.20 \
  --image ghcr.io/siderolabs/installer:v1.9.2 \
  --wait --debug

# Verify
talosctl version --nodes 192.168.1.20
kubectl get nodes -o wide

# Upgrade worker 2
talosctl upgrade --nodes 192.168.1.21 \
  --image ghcr.io/siderolabs/installer:v1.9.2 \
  --wait --debug

# Verify all nodes
talosctl version --nodes 192.168.1.20,192.168.1.21
kubectl get nodes -o wide
```

### Post-Upgrade Verification

```bash
# Full health check after all nodes upgraded
talosctl health --nodes 192.168.1.10
talosctl etcd status --nodes 192.168.1.10
talosctl etcd alarm list --nodes 192.168.1.10
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded
```

---

## Staged Upgrades (--stage)

For environments where you want to control when the reboot happens (e.g., maintenance windows), use `--stage`:

```bash
# Stage the upgrade — downloads image and prepares partition B, but does NOT reboot
talosctl upgrade --nodes 192.168.1.10 \
  --image ghcr.io/siderolabs/installer:v1.9.2 \
  --stage

# The node continues running the current version
# When ready, reboot to activate the staged upgrade:
talosctl reboot --nodes 192.168.1.10
```

**Use cases:**
- Stage upgrades on all nodes during business hours, then reboot one-by-one during the maintenance window
- Pre-download images on nodes with slow network connections
- Coordinate reboots with external automation

---

## Rollback

If a node comes up on the new version but something is wrong, roll back to the previous version:

```bash
# Roll back to the previous OS image (partition A)
talosctl rollback --nodes 192.168.1.10
```

This reboots the node back to the partition it was running before the upgrade. Rollback is available as long as you have not performed another upgrade (which would overwrite the old partition).

**Automatic rollback:** If the upgraded node fails to boot entirely (kernel panic, critical service crash), Talos automatically falls back to the previous partition on the next reboot cycle. No manual intervention required.

**Limitations:**
- Rollback reverts the OS image only — it does not revert etcd data or Kubernetes state
- If etcd schema changed between versions, rolling back a CP node after the new version wrote data may cause issues — this is rare but check release notes
- After a successful rollback, you are back on the old version and can investigate the issue before retrying

---

## Version-Specific Notes

### v1.9.x

- **systemd-udev replaces eudev** — Talos v1.9 switched device management to systemd-udev. Custom udev rules that worked with eudev may need adjustment. Check if you have any machine config entries under `machine.udev`.
- **New `Image Factory` features** — v1.9 added overlay support in Image Factory for SBC platforms.
- **Kubernetes 1.32 support** — v1.9 added support for Kubernetes 1.32. Check the support matrix before upgrading Kubernetes after the Talos upgrade.

### v1.8.x

- **Disk encryption changes** — v1.8 changed the default encryption provider. Read the migration notes if you use disk encryption.
- **Network configuration changes** — Bond and bridge configuration syntax was updated. Verify your machine config patches still apply cleanly.

---

## Monitoring the Upgrade

During upgrade, monitor with:

```bash
# In terminal 1: watch node status
kubectl get nodes -w

# In terminal 2: watch system events
kubectl get events -n kube-system -w

# In terminal 3: watch pod scheduling (for worker upgrades)
kubectl get pods --all-namespaces -o wide -w

# In terminal 4: follow the upgrade progress (if using --debug)
# The --debug flag on talosctl upgrade already streams this

# After each node comes back, verify version
talosctl version --nodes <node-ip>
```

For large clusters, consider scripting the one-at-a-time process:

```bash
#!/usr/bin/env bash
set -euo pipefail

IMAGE="ghcr.io/siderolabs/installer:v1.9.2"
CP_NODES=("192.168.1.10" "192.168.1.11" "192.168.1.12")
WORKER_NODES=("192.168.1.20" "192.168.1.21" "192.168.1.22")

echo "=== Upgrading control-plane nodes ==="
for node in "${CP_NODES[@]}"; do
    echo "--- Upgrading $node ---"
    talosctl upgrade --nodes "$node" --image "$IMAGE" --wait --debug
    echo "--- Verifying $node ---"
    talosctl version --nodes "$node"
    talosctl etcd members --nodes "${CP_NODES[0]}"
    echo "--- $node complete ---"
    echo ""
done

echo "=== Upgrading worker nodes ==="
for node in "${WORKER_NODES[@]}"; do
    echo "--- Upgrading $node ---"
    talosctl upgrade --nodes "$node" --image "$IMAGE" --wait --debug
    echo "--- Verifying $node ---"
    talosctl version --nodes "$node"
    echo "--- $node complete ---"
    echo ""
done

echo "=== Post-upgrade health check ==="
talosctl health --nodes "${CP_NODES[0]}"
kubectl get nodes -o wide
echo "=== Upgrade complete ==="
```
