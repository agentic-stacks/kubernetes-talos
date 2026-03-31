# Ingress on Talos Linux / Bare-Metal Kubernetes

## Overview

Talos Linux clusters run on bare metal (or virtualised bare metal) without cloud
load-balancer integrations.  Two problems must be solved before HTTP traffic can
reach workloads:

1. **External IP allocation** -- something must hand out IP addresses that
   `Service type=LoadBalancer` can bind to.  MetalLB fills this role.
2. **Ingress / Gateway controller** -- something must accept connections on
   those IPs and route them to backend Pods.

This skill covers both layers and the TLS strategy that ties them together.

---

## MetalLB -- LoadBalancer for Bare Metal

### Install with Helm

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update

helm install metallb metallb/metallb \
  --namespace metallb-system \
  --create-namespace \
  --version 0.14.9 \
  --wait
```

Wait for the controller and speaker pods to become ready before applying pool
configuration:

```bash
kubectl -n metallb-system rollout status deploy/metallb-controller
kubectl -n metallb-system rollout status daemonset/metallb-speaker
```

### Layer 2 Mode

Layer 2 is the simplest mode.  The speaker responds to ARP (IPv4) or NDP (IPv6)
requests for allocated IPs on the node network.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.50.100-10.0.50.120   # adjust to your LAN
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
```

Layer 2 limitations: a single node owns each IP, so failover requires gratuitous
ARP which takes a few seconds.  Bandwidth is limited to one node's NIC.

### BGP Mode

BGP mode advertises service IPs via BGP to your ToR switch or router.  Traffic
can be ECMP-balanced across multiple nodes.

```yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: tor-switch
  namespace: metallb-system
spec:
  myASN: 64512
  peerASN: 64513
  peerAddress: 10.0.0.1
  password: "bgp-secret"            # optional MD5 auth
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: bgp-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.60.0/24
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp-adv
  namespace: metallb-system
spec:
  ipAddressPools:
    - bgp-pool
  aggregationLength: 32             # /32 host routes
```

> **Tip:** You can run both L2 and BGP pools simultaneously, assigning different
> pools to different services via `spec.ipAddressPools` selectors.

---

## Ingress Controller Decision Matrix

| Feature | NGINX Ingress | Traefik | Cilium Gateway API | Envoy Gateway |
|---|---|---|---|---|
| API model | Ingress + annotations | Ingress + IngressRoute CRD | Gateway API (HTTPRoute, GRPCRoute, TLSRoute) | Gateway API |
| Protocol support | HTTP/1.1, HTTP/2, gRPC, TCP, UDP | HTTP/1.1, HTTP/2, gRPC, TCP, UDP | HTTP/1.1, HTTP/2, gRPC, TLS passthrough | HTTP/1.1, HTTP/2, gRPC, TCP, UDP, TLS |
| L4/L7 | Both | Both | Both (via Cilium eBPF) | Both |
| mTLS backends | Via annotations | Via middleware | Native (Cilium identity) | Via BackendTLSPolicy |
| Rate limiting | annotation-based | Middleware CRD | HTTPRoute filter (ext proc) | BackendTrafficPolicy / RateLimitFilter |
| WAF / ModSecurity | Supported | Plugin (enterprise) | No | ext_authz / Wasm |
| Dashboard | None (Prometheus metrics) | Built-in | Hubble UI | Envoy admin (debug) |
| Talos-specific notes | Well-tested, large community | Good, ships as k3s default | Zero extra pods when Cilium is the CNI | Standalone, good for multi-tenant |
| Maturity | GA, very stable | GA, stable | GA since Cilium 1.15 | GA since v1.1 |

**Recommendations:**

- **Default choice:** NGINX Ingress -- widest community support and documentation.
- **Already running Cilium CNI:** Cilium Gateway API -- no extra data-plane pods.
- **Advanced middleware / dashboard:** Traefik.
- **Multi-tenant or Envoy ecosystem:** Envoy Gateway.

---

## TLS Strategy Overview

All ingress controllers integrate with **cert-manager** for automatic TLS:

1. **cert-manager** watches `Ingress` or `Gateway` resources for TLS
   annotations/references.
2. It creates an ACME `Order` (Let's Encrypt) or signs from an internal CA.
3. The resulting certificate is stored in a Kubernetes `Secret` of type
   `kubernetes.io/tls`.
4. The ingress controller mounts the secret and terminates TLS.

Supported flows:

| Flow | Challenge | Use case |
|---|---|---|
| Let's Encrypt HTTP-01 | Ingress controller serves `/.well-known/acme-challenge` | Public-facing services, simple setup |
| Let's Encrypt DNS-01 | cert-manager creates a TXT record via cloud DNS API | Wildcard certs, private clusters |
| Self-signed CA | cert-manager generates a CA and signs certs | Dev / air-gapped environments |
| Vault PKI | cert-manager talks to HashiCorp Vault | Enterprise PKI |

See [cert-manager.md](cert-manager.md) for full installation and configuration.

---

## File Index

| File | Contents |
|---|---|
| [nginx.md](nginx.md) | NGINX Ingress Controller install, config, examples |
| [traefik.md](traefik.md) | Traefik install, IngressRoute CRDs, middleware |
| [cilium-gateway-api.md](cilium-gateway-api.md) | Cilium Gateway API setup, HTTPRoute, GRPCRoute |
| [cert-manager.md](cert-manager.md) | cert-manager install, ClusterIssuers, trust-manager |
