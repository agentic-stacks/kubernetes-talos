# Foundation — Infrastructure

Platform-specific guidance for provisioning machines that will run Talos Linux. This skill covers every supported deployment target from bare metal and local virtualization through public cloud providers.

---

## 1. Choosing a Platform

Select your platform based on these criteria:

| Criterion | Bare Metal | Proxmox / libvirt | VMware | AWS / GCP / Azure | Hetzner / DO |
|---|---|---|---|---|---|
| Cost at scale | Lowest | Low (CapEx) | Medium–High | High (OpEx) | Medium |
| Control over hardware | Full | Full | Full | None | Partial |
| Network performance | Highest | High | High | Variable | Good |
| Operational complexity | High | Medium | Medium | Low–Medium | Low |
| Cloud provider integration | Manual | Manual | vSphere CCM | Native CCM | CCM available |
| Best for | Production on-prem | Homelabs, dev clusters | Enterprise on-prem | Cloud-native workloads | Cost-effective cloud |

**Decision rules:**

- If you already own servers and need maximum performance → **bare-metal**
- If you have a Proxmox or KVM hypervisor for dev/staging → **proxmox** or **libvirt**
- If your organization is on vSphere → **vmware**
- If you need elastic scaling and cloud integrations → **aws**, **gcp**, or **azure**
- If you want cost-effective cloud without major vendor lock-in → **hetzner** or **digital-ocean**

---

## 2. Common Prerequisites

### Required Tools

```bash
# Install talosctl (must match target Talos version)
curl -sL https://talos.dev/install | sh

# Verify
talosctl version --client
```

Talos 1.9.x pairs with Kubernetes 1.32.x. Talos 1.8.x pairs with Kubernetes 1.31.x. See `reference/compatibility` for the full matrix.

### Network Ports

Every Talos node (control plane and worker) requires these ports accessible between nodes and from your management workstation:

| Port | Protocol | Purpose | Required By |
|---|---|---|---|
| 50000 | TCP | Talos API (talosctl) | Management workstation → all nodes |
| 50001 | TCP | Talos trustd (cert distribution) | Control plane nodes only |
| 6443 | TCP | Kubernetes API server | All nodes → control plane endpoint |
| 2379 | TCP | etcd client API | Control plane nodes only |
| 2380 | TCP | etcd peer communication | Control plane nodes only |
| 10250 | TCP | kubelet API | Intra-cluster |
| 179 | TCP | BGP (if using Calico/Cilium BGP) | Optional |
| 4240 | TCP | Cilium health check | Optional, if using Cilium |
| 8472 | UDP | VXLAN (Flannel/Cilium overlay) | Optional, if using overlay |
| 51820 | UDP | WireGuard (Cilium WireGuard mode) | Optional |

Talos does not expose SSH (port 22) — there is no shell access.

### DNS Planning

Plan DNS before provisioning:

```
# Control plane endpoint (load balancer or VIP)
k8s.example.com          → 10.0.0.10  (LB/VIP/single CP IP)

# Individual nodes (optional but recommended)
cp1.example.com          → 10.0.0.11
cp2.example.com          → 10.0.0.12
cp3.example.com          → 10.0.0.13
worker1.example.com      → 10.0.0.21
worker2.example.com      → 10.0.0.22
```

The control plane endpoint (`k8s.example.com`) is what you pass to `talosctl gen config` as `--endpoint`. It must remain stable after cluster creation — changing it requires regenerating and reapplying machine configs.

### IP Address Planning

```
Management subnet:   10.0.0.0/24
  Control plane VIP: 10.0.0.10  (reserved, not assigned to any host OS)
  Control plane 1:   10.0.0.11
  Control plane 2:   10.0.0.12
  Control plane 3:   10.0.0.13
  Workers:           10.0.0.21–10.0.0.99

Pod CIDR:            10.244.0.0/16  (default, adjust to avoid overlap)
Service CIDR:        10.96.0.0/12   (default, adjust to avoid overlap)
```

Verify your chosen pod and service CIDRs do not overlap with your physical network. These are set in the machine config and cannot be changed after cluster creation without rebuilding.

---

## 3. Image Provisioning Overview

Talos images are obtained from the **Image Factory** at `factory.talos.dev` or built with `talosctl`.

### Image Types

| Format | Use Case |
|---|---|
| ISO | Boot from virtual or physical optical drive; installs Talos to disk |
| PXE / iPXE | Network boot for bare metal; no physical media needed |
| Raw disk image | Write directly to disk; used for pre-provisioned bare metal |
| qcow2 | QEMU/KVM virtual machines (Proxmox, libvirt) |
| VMDK | VMware ESXi virtual machines |
| OVA | VMware vSphere (packaged VMDK + descriptor) |
| AMI | AWS EC2 — provided by Siderolabs, searchable by account ID |
| GCE image | Google Compute Engine — upload or use community images |
| Azure VHD | Azure — upload to managed disk or use Marketplace |
| cloud image | Generic cloud-init capable image |

### Image Factory

The Image Factory generates custom Talos images with specific system extensions baked in (e.g., QEMU guest agent, iSCSI initiator, NVIDIA drivers):

```bash
# Visit factory.talos.dev to generate a schematic ID for your extensions
# Example: bare metal with Intel/AMD microcode updates and iSCSI support
SCHEMATIC_ID="your-schematic-id-from-factory"
TALOS_VERSION="v1.9.5"

# Download ISO
curl -LO "https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/metal-amd64.iso"

# Download qcow2
curl -LO "https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/metal-amd64.qcow2"

# Download AWS AMI (register it yourself) or use raw for other clouds
curl -LO "https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/metal-amd64.raw.xz"
```

### Official Images (No Extensions)

When you do not need extensions, use official release images directly:

```bash
TALOS_VERSION="v1.9.5"

# ISO
https://github.com/siderolabs/talos/releases/download/${TALOS_VERSION}/metal-amd64.iso

# qcow2
https://github.com/siderolabs/talos/releases/download/${TALOS_VERSION}/metal-amd64.qcow2.xz

# VMDK
https://github.com/siderolabs/talos/releases/download/${TALOS_VERSION}/vmware-amd64.vmdk.xz

# OVA
https://github.com/siderolabs/talos/releases/download/${TALOS_VERSION}/vmware-amd64.ova
```

---

## 4. Platform Index

| Platform | Guide | Provisioning Method | CP Endpoint Strategy |
|---|---|---|---|
| Bare Metal | [bare-metal.md](./bare-metal.md) | PXE/iPXE or ISO | Talos VIP |
| Proxmox VE | [proxmox.md](./proxmox.md) | ISO or cloud image via `qm` | VIP or HAProxy |
| VMware vSphere | [vmware.md](./vmware.md) | OVA import via `govc` | vSphere LB or VIP |
| libvirt / KVM | [libvirt.md](./libvirt.md) | qcow2 via `virt-install` | Talos VIP |
| AWS EC2 | [aws.md](./aws.md) | Official AMIs | AWS NLB |
| Google Cloud | [gcp.md](./gcp.md) | GCE images | Internal TCP LB |
| Azure | [azure.md](./azure.md) | Marketplace image or VHD | Azure Standard LB |
| Hetzner Cloud | [hetzner.md](./hetzner.md) | Custom image via `hcloud` | Hetzner LB or floating IP |
| DigitalOcean | [digital-ocean.md](./digital-ocean.md) | Custom image via `doctl` | DO Load Balancer |

---

## Next Step

After provisioning infrastructure, generate machine configs:

```
foundation/machine-config → deploy/bootstrap
```
