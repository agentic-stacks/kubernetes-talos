# Bare Metal Provisioning

Talos Linux on physical servers. The most powerful deployment target — full hardware access, no hypervisor overhead, and support for SR-IOV, DPDK, and direct NVMe access.

---

## 1. Prerequisites

**Hardware:**
- x86_64 or arm64 servers (Talos supports both architectures)
- Minimum 2 CPU cores, 2 GB RAM per node (4 cores / 8 GB recommended for control plane)
- One dedicated disk for Talos installation (separate from any data disks)
- IPMI, iDRAC, iLO, or BMC for out-of-band management (strongly recommended)
- Network switch with VLAN support (recommended for pod/service network isolation)

**Management workstation:**
- `talosctl` installed and matching target Talos version
- Network access to all nodes on port 50000 (Talos API)
- Access to BMC/IPMI interfaces for console access during initial boot

**Optional but recommended:**
- `matchbox` server for PXE provisioning automation
- DHCP server with static reservations for node IPs
- DNS entries for all nodes and the control plane endpoint

---

## 2. Image Provisioning

### Option A: ISO Boot (Simplest)

Download the ISO and write it to a USB drive or mount it via virtual media through your BMC:

```bash
TALOS_VERSION="v1.9.5"

# Download official metal ISO
curl -LO "https://github.com/siderolabs/talos/releases/download/${TALOS_VERSION}/metal-amd64.iso"

# Write to USB (replace /dev/sdX with your USB device)
sudo dd if=metal-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

Boot the server from the ISO. Talos boots into maintenance mode and awaits a machine config via the API. The node is not yet configured — you apply the machine config after booting.

### Option B: PXE / iPXE Boot (Scalable)

PXE boot is preferred for fleets because it requires no physical media and integrates with provisioning automation.

**Set up a matchbox server** (runs on any Linux host):

```bash
# Download matchbox
MATCHBOX_VERSION="v0.10.0"
curl -LO "https://github.com/poseidon/matchbox/releases/download/${MATCHBOX_VERSION}/matchbox-linux-amd64.tar.gz"
tar xzf matchbox-linux-amd64.tar.gz
sudo mv matchbox-linux-amd64/matchbox /usr/local/bin/

# Create asset directory and download Talos kernel + initrd
mkdir -p /var/lib/matchbox/assets/talos/${TALOS_VERSION}
cd /var/lib/matchbox/assets/talos/${TALOS_VERSION}
curl -LO "https://github.com/siderolabs/talos/releases/download/${TALOS_VERSION}/vmlinuz-amd64"
curl -LO "https://github.com/siderolabs/talos/releases/download/${TALOS_VERSION}/initramfs-amd64.xz"

# Start matchbox
matchbox -address=0.0.0.0:8080 -rpc-address=0.0.0.0:8081 \
  -assets-path=/var/lib/matchbox/assets
```

**Create a matchbox group and profile for control plane nodes:**

```json
// /var/lib/matchbox/profiles/control-plane.json
{
  "id": "control-plane",
  "name": "Talos Control Plane",
  "boot": {
    "kernel": "/assets/talos/v1.9.5/vmlinuz-amd64",
    "initrd": ["/assets/talos/v1.9.5/initramfs-amd64.xz"],
    "args": [
      "initrd=initramfs-amd64.xz",
      "init_on_alloc=1",
      "slab_nomerge",
      "pti=on",
      "console=tty0",
      "console=ttyS0,115200n8",
      "talos.platform=metal",
      "talos.config=http://matchbox.example.com:8080/config?mac=${mac}"
    ]
  }
}
```

```json
// /var/lib/matchbox/groups/cp1.json
{
  "id": "cp1",
  "name": "Control Plane Node 1",
  "profile": "control-plane",
  "selector": {
    "mac": "aa:bb:cc:dd:ee:01"
  },
  "metadata": {
    "talos_config_url": "http://matchbox.example.com:8080/generic?mac=${mac}"
  }
}
```

**Configure your DHCP server** to point PXE clients to matchbox (example for dnsmasq):

```ini
# /etc/dnsmasq.conf additions
dhcp-range=10.0.0.100,10.0.0.200,12h
dhcp-boot=tag:ipxe,http://matchbox.example.com:8080/boot.ipxe
dhcp-match=set:ipxe,175  # iPXE sends option 175
enable-tftp
tftp-root=/var/lib/tftpboot
dhcp-boot=undionly.kpxe
```

### Option C: Custom Image with Extensions (Image Factory)

Use this when nodes need system extensions (e.g., iSCSI initiator for Longhorn, Intel microcode updates, DRBD):

```bash
# 1. Go to factory.talos.dev and select your extensions
# 2. Copy the generated schematic ID
SCHEMATIC_ID="376567988ad370138ad8f2f4b52a82c8ce81c5c3ae3f5d2e1b15e3c7d6a34f8"
TALOS_VERSION="v1.9.5"

# Download custom ISO
curl -LO "https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/metal-amd64.iso"

# Or download kernel + initrd for PXE
curl -LO "https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/kernel-amd64"
curl -LO "https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/initramfs-amd64.xz"
```

Store the schematic ID in your cluster repository — you will need it for upgrades to ensure extensions persist.

---

## 3. Network Requirements

| Port | Protocol | Direction | Purpose |
|---|---|---|---|
| 50000 | TCP | Management → nodes | Talos API |
| 50001 | TCP | CP ↔ CP | Talos trustd |
| 6443 | TCP | All → CP endpoint | Kubernetes API |
| 2379–2380 | TCP | CP ↔ CP | etcd |
| 10250 | TCP | Intra-cluster | kubelet |

**NIC bonding / LACP** is supported via machine config. Configure it in the `machine.network.interfaces` section with `bond` type.

**VLAN tagging** is supported natively in Talos machine config — no kernel modules required.

---

## 4. Machine Config Specifics

### Install Disk

Identify the correct install disk path. On most servers:

```
/dev/sda      — SATA/SAS HDD or SSD
/dev/nvme0n1  — NVMe SSD (first NVMe device)
/dev/vda      — Virtio disk (not typical on bare metal)
```

To discover the disk name before applying config, boot from ISO and check:

```bash
# From your workstation, query the maintenance-mode API
talosctl -n 10.0.0.11 --talosconfig /dev/null \
  get disks --insecure
```

**Machine config patch for install disk:**

```yaml
# patches/install-disk.yaml
machine:
  install:
    disk: /dev/nvme0n1
    image: ghcr.io/siderolabs/installer:v1.9.5
    wipe: false
```

If using Image Factory extensions, use the factory installer image:

```yaml
machine:
  install:
    disk: /dev/nvme0n1
    image: factory.talos.dev/installer/${SCHEMATIC_ID}:v1.9.5
    wipe: false
```

### VIP Configuration (Control Plane HA)

Add VIP configuration to all control plane node machine configs:

```yaml
# patches/cp-vip.yaml
machine:
  network:
    interfaces:
      - interface: eth0
        addresses:
          - 10.0.0.11/24  # This node's static IP
        vip:
          ip: 10.0.0.10   # Shared VIP — only one CP node holds this at a time
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.0.1
```

The VIP uses ARP-based failover. Adjust `interface` to match your NIC name (check with `talosctl get links --insecure` during initial boot).

### BIOS/UEFI Settings

Talos requires:
- Secure Boot: **disabled** (Talos does not support Secure Boot by default)
- Legacy boot: supported, but UEFI preferred
- CPU virtualization extensions (VT-x / AMD-V): enabled if running VMs on the nodes

---

## 5. Control Plane Endpoint

**Talos VIP** is the recommended approach for bare metal HA. It requires no external load balancer.

The VIP floats between healthy control plane nodes using ARP. If the current VIP holder fails, another control plane node takes over within a few seconds.

Requirements:
- All control plane nodes on the same L2 network segment
- VIP IP must not be assigned to any node as a static IP
- VIP IP must be in the same subnet as the control plane NICs

Alternative: **keepalived** (external, not managed by Talos):

```bash
# Install keepalived on each CP node's host OS — not applicable for Talos
# For Talos, use the built-in VIP instead
```

For multi-rack deployments where control plane nodes span L3 boundaries, use an external load balancer (hardware LB, HAProxy, or cloud LB) instead of VIP.

---

## 6. Provisioning Commands

### Step 1: Generate Machine Configs

```bash
# Create your cluster directory
mkdir my-cluster && cd my-cluster

# Generate configs with VIP as control plane endpoint
talosctl gen config my-cluster https://10.0.0.10:6443 \
  --output-dir . \
  --config-patch-control-plane @patches/cp-vip.yaml \
  --config-patch-control-plane @patches/install-disk.yaml \
  --config-patch @patches/install-disk.yaml

# Preserve originals before any modifications
cp controlplane.yaml controlplane.yaml.orig
cp worker.yaml worker.yaml.orig
```

### Step 2: Validate Configs

```bash
talosctl validate --config controlplane.yaml --mode metal
talosctl validate --config worker.yaml --mode metal
```

### Step 3: Apply Config to Control Plane Nodes

Nodes must be booted from Talos ISO or PXE and in maintenance mode (no config applied yet):

```bash
# Apply to first control plane node
talosctl apply-config \
  --nodes 10.0.0.11 \
  --file controlplane.yaml \
  --insecure   # Required before PKI is established

# Apply to remaining control plane nodes
talosctl apply-config --nodes 10.0.0.12 --file controlplane.yaml --insecure
talosctl apply-config --nodes 10.0.0.13 --file controlplane.yaml --insecure
```

After config is applied, nodes will install Talos to disk and reboot. Wait for them to come up (watch console via IPMI/BMC).

### Step 4: Bootstrap etcd

Bootstrap etcd on exactly one control plane node — only do this once:

```bash
# Set up talosconfig
export TALOSCONFIG=./talosconfig
talosctl config endpoint 10.0.0.11 10.0.0.12 10.0.0.13
talosctl config node 10.0.0.11

# Bootstrap etcd on the first CP node only
talosctl bootstrap --nodes 10.0.0.11
```

Watch bootstrap progress:

```bash
talosctl dmesg --follow --nodes 10.0.0.11
```

### Step 5: Apply Config to Worker Nodes

```bash
talosctl apply-config --nodes 10.0.0.21 --file worker.yaml --insecure
talosctl apply-config --nodes 10.0.0.22 --file worker.yaml --insecure
talosctl apply-config --nodes 10.0.0.23 --file worker.yaml --insecure
```

### Step 6: Retrieve kubeconfig

```bash
# Wait for Kubernetes API to become available (~2–3 minutes after bootstrap)
talosctl kubeconfig --nodes 10.0.0.11 --force

# Verify cluster
kubectl get nodes
kubectl get pods -A
```

---

## 7. Platform-Specific Gotchas

**Disk detection race on fast NVMe:** On some servers with fast NVMe drives, Talos may not correctly identify the install disk if `/dev/nvme0n1` is not present at install time. Specify the disk by ID instead:

```yaml
machine:
  install:
    disk: /dev/disk/by-id/nvme-Samsung_SSD_980_PRO_2TB_XXXXXXXXX
```

**RAID controllers:** Hardware RAID controllers present disks as `/dev/sda` regardless of underlying drive type. Software RAID (Linux MD) is not supported in Talos — use individual disks or hardware RAID.

**Secure Boot:** Talos does not support Secure Boot in standard builds. If your security policy requires Secure Boot, contact Siderolabs for Enterprise support or use the Talos UEFI Secure Boot experimental feature (Talos 1.9+, not production-ready).

**IPMI/BMC serial console:** Set `console=ttyS0,115200n8` in your PXE kernel args to get serial console output through IPMI. Without this, the console output only goes to the local VGA.

**VIP and bond interfaces:** When using NIC bonding with a VIP, specify the VIP on the bond interface (e.g., `bond0`), not on individual member interfaces.

**MTU for jumbo frames:** If your network supports jumbo frames and you want to use them, set MTU in the machine config:

```yaml
machine:
  network:
    interfaces:
      - interface: eth0
        mtu: 9000
```

**First boot time:** Bare metal installs can take 3–10 minutes depending on disk speed and the size of the installer image. Do not reboot prematurely. Watch via BMC console.

**matchbox and machine config serving:** matchbox can serve Talos machine configs via its HTTP API. This allows fully automated provisioning where nodes request their own configs at boot time using their MAC address.

**iSCSI for storage (Longhorn, OpenEBS):** If your storage solution requires iSCSI (e.g., Longhorn on some configurations), you must include the `siderolabs/iscsi-tools` extension in your Image Factory schematic before provisioning. This cannot be added after the fact without an upgrade.
