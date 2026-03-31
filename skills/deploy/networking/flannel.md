# Flannel — Talos Default CNI

Flannel is the default CNI included with Talos Linux. It provides basic L3 networking with VXLAN overlay.

## When to Choose Flannel

- Simplest possible networking — zero additional configuration
- Small clusters or development environments
- No requirement for NetworkPolicy enforcement
- No need for kube-proxy replacement
- Getting started quickly without CNI decisions

## Machine Config

Flannel is the default. Either omit the CNI configuration or explicitly set it:

```yaml
cluster:
  network:
    cni:
      name: flannel
    podSubnets:
      - 10.244.0.0/16
    serviceSubnets:
      - 10.96.0.0/12
```

## How It Works

- Talos deploys Flannel automatically during bootstrap
- Uses VXLAN backend by default
- Assigns a /24 subnet per node from the podSubnets range
- No manual installation required — it is part of the Talos bootstrap process

## Verification

After bootstrap:

```bash
kubectl get pods -n kube-system -l app=flannel
# All flannel pods should be Running

kubectl get nodes
# All nodes should be Ready
```

## Limitations

- **No NetworkPolicy**: Flannel does not implement NetworkPolicy. If you need NetworkPolicy enforcement, use Cilium or Calico, or add a separate policy engine (e.g., Calico policy-only mode).
- **No kube-proxy replacement**: Flannel does not replace kube-proxy. kube-proxy runs as a DaemonSet alongside Flannel.
- **No encryption**: Flannel's VXLAN does not encrypt traffic between nodes. For in-transit encryption, use a service mesh or choose Cilium/Calico with WireGuard.
- **No L7 visibility**: No deep packet inspection or L7 network policies.

## Switching Away from Flannel

Switching CNI after cluster creation is disruptive and not officially supported. If you anticipate needing features like NetworkPolicy, kube-proxy replacement, or encryption, choose Cilium or Calico from the start by setting `cni.name: none` before bootstrap.
