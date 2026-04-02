# VMware vSphere / ESXi

## Prerequisites

- vSphere 7.0+ or standalone ESXi
- `govc` CLI installed and configured (`GOVC_URL`, `GOVC_USERNAME`, `GOVC_PASSWORD`, `GOVC_INSECURE`)
- Resource pool or cluster with sufficient capacity
- Distributed or standard port group for Talos networking
- Talos OVA downloaded

## Image Provisioning

Download the VMware OVA from Image Factory:

```bash
wget https://factory.talos.dev/image/<schematic-id>/v1.9.0/vmware-amd64.ova
```

Import the OVA as a template:

```bash
govc import.ova -name talos-template -pool /Datacenter/host/Cluster/Resources talos-amd64.ova
govc vm.markastemplate talos-template
```

## Network Requirements

- VMXNET3 adapter (default in OVA)
- Port group with DHCP or static IPs planned
- Promiscuous mode enabled on port group if using Talos VIP
- Firewall rules: TCP 50000, 50001, 6443, 2379-2380, 10250

## Machine Config Specifics

```yaml
machine:
  install:
    disk: /dev/sda
    image: ghcr.io/siderolabs/installer:v1.9.0
  network:
    interfaces:
      - interface: eth0
        dhcp: true
        # or static:
        # addresses:
        #   - 10.0.0.11/24
        # routes:
        #   - network: 0.0.0.0/0
        #     gateway: 10.0.0.1
```

For vSphere Cloud Provider integration, add cloud config:

```yaml
machine:
  kubelet:
    extraArgs:
      cloud-provider: external
cluster:
  externalCloudProvider:
    enabled: true
    manifests:
      - https://raw.githubusercontent.com/kubernetes/cloud-provider-vsphere/master/releases/latest/vsphere-cloud-controller-manager.yaml
```

## Control Plane Endpoint

**Option 1: Talos VIP** — requires promiscuous mode on port group.

**Option 2: NSX Load Balancer** — if using NSX-T, create an L4 load balancer for TCP 6443.

**Option 3: HAProxy Appliance** — deploy the vSphere HAProxy appliance for load balancing.

## Provisioning Commands

```bash
# Clone from template
govc vm.clone -vm talos-template -on=false -c 4 -m 8192 talos-cp1
govc vm.clone -vm talos-template -on=false -c 4 -m 8192 talos-cp2
govc vm.clone -vm talos-template -on=false -c 4 -m 8192 talos-cp3
govc vm.clone -vm talos-template -on=false -c 2 -m 4096 talos-worker1

# Resize disks
govc vm.disk.change -vm talos-cp1 -size 40G
govc vm.disk.change -vm talos-worker1 -size 100G

# Power on
govc vm.power -on talos-cp1 talos-cp2 talos-cp3 talos-worker1

# Get IPs
govc vm.ip talos-cp1
```

## Platform-Specific Gotchas

- **Promiscuous mode**: Required for Talos VIP on standard vSwitches. Not needed with NSX or external LB.
- **Disk controller**: OVA uses pvscsi by default. Disk shows as `/dev/sda`.
- **Open VM Tools**: Talos includes a minimal VMware tools implementation. It reports IP addresses but does not support all guest operations.
- **vSphere CSI**: For persistent storage, install the vSphere CSI driver (separate from this provisioning guide — see deploy/storage).
- **DRS**: If using DRS, create anti-affinity rules to spread control plane nodes across hosts.
- **Snapshots**: Avoid VMware snapshots on control plane nodes — etcd does not tolerate disk I/O pauses. Use `talosctl etcd snapshot` instead.
