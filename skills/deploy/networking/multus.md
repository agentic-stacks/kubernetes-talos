# Multus CNI — Multi-Network Pods

Multus is a meta-plugin that enables attaching multiple network interfaces to pods. It delegates to a primary CNI (Cilium, Flannel, Calico) and adds secondary interfaces.

## When to Use Multus

- Pods require multiple network interfaces (e.g., data plane + management plane)
- SR-IOV for high-performance networking (telco, HPC)
- DPDK workloads
- Network function virtualization (NFV)
- Bridging to specific VLANs or physical networks

Most clusters do NOT need Multus. Only deploy it when you have a specific multi-network requirement.

## Machine Config Prerequisites

Multus works alongside any primary CNI. Set up your primary CNI first (Cilium, Flannel, or Calico), then install Multus.

For SR-IOV, you may need the `sriov` Talos system extension and appropriate hardware.

## Installation

**Using the thick plugin (Recommended for Talos):**

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
```

The thick plugin runs as a DaemonSet and does not require writing to the host CNI directory (compatible with Talos immutable filesystem).

## Network Attachment Definitions

Define additional networks as `NetworkAttachmentDefinition` resources:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-net
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.2.0/24",
        "rangeStart": "192.168.2.100",
        "rangeEnd": "192.168.2.200"
      }
    }
```

Attach to pods via annotation:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-net-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-net
spec:
  containers:
    - name: app
      image: nginx
```

The pod gets its primary interface from the primary CNI and an additional `net1` interface from the macvlan NetworkAttachmentDefinition.

## Talos-Specific Gotchas

- **Thick plugin only**: Use the thick (DaemonSet-based) Multus deployment. The thin plugin requires writing to host filesystem paths that Talos does not expose.
- **SR-IOV**: Requires hardware support (SR-IOV capable NIC) and the `sriov` Talos system extension. Not available on all platforms.
- **Interface naming**: Talos v1.9 uses `systemd-udev` for interface naming. Verify physical interface names before configuring macvlan/ipvlan masters.
