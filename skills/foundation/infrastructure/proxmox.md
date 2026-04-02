# Proxmox VE

## Prerequisites

- Proxmox VE 7.x or 8.x
- At least one VM host with sufficient resources (4 vCPU, 8 GB RAM per Talos node recommended)
- Network bridge configured (default: `vmbr0`)
- Talos ISO or cloud image downloaded
- `qm` CLI or Proxmox web UI access

## Image Provisioning

**Option 1: ISO Install**

Download the Talos metal ISO from Image Factory:

```bash
# Standard ISO
wget https://factory.talos.dev/image/<schematic-id>/v1.9.0/metal-amd64.iso

# Or the default ISO (no extensions)
wget https://github.com/siderolabs/talos/releases/download/v1.9.0/metal-amd64.iso
```

Upload to Proxmox storage:

```bash
# Via CLI
scp metal-amd64.iso root@proxmox:/var/lib/vz/template/iso/

# Or via the web UI: Datacenter > Storage > ISO Images > Upload
```

**Option 2: Cloud Image (Recommended for Templates)**

Download the nocloud qcow2 image:

```bash
wget https://factory.talos.dev/image/<schematic-id>/v1.9.0/nocloud-amd64.qcow2
```

Import as a VM disk to create a template.

## Network Requirements

- Bridge networking (`vmbr0` or custom bridge)
- VLAN tagging supported via Proxmox bridge configuration
- VIP works via Talos native VIP (gratuitous ARP on the bridge network)
- Firewall rules: allow TCP 50000, 50001, 6443, 2379-2380, 10250 between Talos VMs

## Machine Config Specifics

```yaml
machine:
  install:
    disk: /dev/sda      # virtio disk
    image: ghcr.io/siderolabs/installer:v1.9.0
    extensions:
      - image: ghcr.io/siderolabs/qemu-guest-agent:v1.9.0
```

The **QEMU guest agent extension** is strongly recommended — it enables Proxmox to report VM IP addresses, perform graceful shutdowns, and freeze filesystems for snapshots. Include it in your Image Factory schematic:

```yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/qemu-guest-agent
```

## Control Plane Endpoint

**Option 1: Talos VIP (Simplest)**

Works on Proxmox bridge networks (L2). Configure in machine config:

```yaml
machine:
  network:
    interfaces:
      - interface: eth0
        dhcp: true
        vip:
          ip: 192.168.1.100
```

**Option 2: HAProxy on Proxmox Host**

Run HAProxy as an LXC container on Proxmox, load-balancing TCP 6443 to control plane nodes.

## Provisioning Commands

**Create a VM from ISO:**

```bash
# Create VM
qm create 100 \
  --name talos-cp1 \
  --memory 4096 \
  --cores 4 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-pci \
  --scsi0 local-lvm:32 \
  --cdrom local:iso/metal-amd64.iso \
  --boot order=scsi0;ide2 \
  --ostype l26 \
  --agent enabled=1

# Start VM
qm start 100
```

**Create a template from cloud image:**

```bash
# Create VM shell
qm create 9000 \
  --name talos-template \
  --memory 4096 \
  --cores 4 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-pci \
  --agent enabled=1 \
  --ostype l26

# Import the cloud image as the boot disk
qm importdisk 9000 nocloud-amd64.qcow2 local-lvm
qm set 9000 --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --boot order=scsi0

# Convert to template
qm template 9000

# Clone for each node
qm clone 9000 101 --name talos-cp1 --full
qm clone 9000 102 --name talos-cp2 --full
qm clone 9000 103 --name talos-cp3 --full

# Resize disks as needed
qm resize 101 scsi0 +28G
```

## Platform-Specific Gotchas

- **QEMU guest agent**: Without the extension, Proxmox cannot detect VM IPs or perform graceful shutdown. Always include it.
- **Disk type**: Use `virtio-scsi-pci` controller with `scsi0` for best performance. Avoid IDE.
- **CPU type**: Set `--cpu host` for best performance if not migrating between different CPU architectures.
- **Serial console**: Add `--serial0 socket` and `--vga serial0` to access the Talos console via Proxmox's serial terminal (useful for initial setup and troubleshooting).
- **Cloud-init drive**: When using the nocloud image, Proxmox's built-in cloud-init drive is NOT used for Talos. Talos uses its own machine config mechanism via `talosctl apply-config`.
- **Storage backends**: Proxmox local-lvm (LVM-thin) works well. Ceph-backed storage also supported but adds latency for etcd.
