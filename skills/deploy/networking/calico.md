# Calico on Talos

Calico provides L3 networking with NetworkPolicy enforcement. On Talos, use eBPF dataplane mode for best compatibility since Talos does not allow workloads to load kernel modules.

## When to Choose Calico

- Existing Calico expertise on the team
- Need L3-L4 NetworkPolicy enforcement
- Enterprise support via Tigera
- BGP peering requirements for on-premise networks

## Machine Config Prerequisites

Set CNI to none and optionally disable kube-proxy (for eBPF mode):

```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true    # required for eBPF kube-proxy replacement
```

## Installation

**Using the Tigera Operator (Recommended):**

```bash
# Install the operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

# Create the Installation CR
cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - cidr: 10.244.0.0/16
        encapsulation: VXLAN
        natOutgoing: Enabled
        nodeSelector: all()
    linuxDataplane: BPF
  cni:
    type: Calico
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

**Important**: Set `linuxDataplane: BPF` for Talos compatibility. The iptables dataplane requires kernel module loading which Talos does not allow from workloads.

## Verification

```bash
# Check Calico pods
kubectl get pods -n calico-system

# Check nodes are Ready
kubectl get nodes

# Verify eBPF mode
kubectl get installation default -o jsonpath='{.spec.calicoNetwork.linuxDataplane}'
# Should output: BPF
```

## NetworkPolicy

Calico supports Kubernetes NetworkPolicy (L3-L4) and its own GlobalNetworkPolicy CRD for cluster-wide rules:

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-all-default
spec:
  selector: projectcalico.org/namespace == "default"
  types:
    - Ingress
    - Egress
```

## Talos-Specific Gotchas

- **eBPF mode required**: The iptables dataplane will not work reliably on Talos. Always use `linuxDataplane: BPF`.
- **kube-proxy replacement**: When using eBPF mode, Calico replaces kube-proxy. Set `cluster.proxy.disabled: true` in machine config before bootstrap.
- **Resource overhead**: Calico operator + node DaemonSet + typha (for scale). Similar resource footprint to Cilium.
- **Wireguard encryption**: Supported in eBPF mode. Enable via `calicoNetwork.linuxDataplane: BPF` and `spec.calicoNetwork.bgp: Disabled` with VXLAN encapsulation.
