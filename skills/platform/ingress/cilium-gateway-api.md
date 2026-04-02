# Cilium Gateway API

## Prerequisites

Cilium must be installed (or upgraded) with Gateway API support enabled.  On a
Talos cluster where Cilium is already the CNI, upgrade via Helm:

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set gatewayAPI.enabled=true \
  --set kubeProxyReplacement=true \
  --set l7Proxy=true \
  --wait
```

Cilium automatically installs the Gateway API CRDs when `gatewayAPI.enabled=true`.
Verify:

```bash
kubectl get crd gateways.gateway.networking.k8s.io
kubectl get crd httproutes.gateway.networking.k8s.io
kubectl get crd grpcroutes.gateway.networking.k8s.io
kubectl get crd tlsroutes.gateway.networking.k8s.io
```

Cilium uses its own Envoy proxy embedded in the Cilium agent -- no separate
ingress controller pods are created.

---

## GatewayClass

Cilium registers a `GatewayClass` named `cilium`.  Confirm it exists:

```bash
kubectl get gatewayclass cilium
```

If it is missing (rare), create it:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
```

---

## Gateway

A `Gateway` allocates a listener (virtual server) on a LoadBalancer IP.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: default
spec:
  gatewayClassName: cilium
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: wildcard-example-com-tls
            namespace: default
      allowedRoutes:
        namespaces:
          from: All
```

MetalLB will assign an external IP to the resulting `Service type=LoadBalancer`.

```bash
kubectl get gateway main-gateway
kubectl get svc -l io.cilium.gateway/owning-gateway=main-gateway
```

---

## HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
  namespace: default
spec:
  parentRefs:
    - name: main-gateway
      sectionName: https
  hostnames:
    - app.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: app-svc
          port: 80
          weight: 100
```

### HTTPRoute with Header-Based Routing and Request Redirect

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: advanced-route
  namespace: default
spec:
  parentRefs:
    - name: main-gateway
      sectionName: https
  hostnames:
    - api.example.com
  rules:
    # Canary: route 10% of traffic with header x-canary: true to v2
    - matches:
        - headers:
            - name: x-canary
              value: "true"
      backendRefs:
        - name: api-v2
          port: 80
    # Default
    - matches:
        - path:
            type: PathPrefix
            value: /v1
      backendRefs:
        - name: api-v1
          port: 80
    # Redirect /old to /v1
    - matches:
        - path:
            type: Exact
            value: /old
      filters:
        - type: RequestRedirect
          requestRedirect:
            path:
              type: ReplaceFullPath
              replaceFullPath: /v1
            statusCode: 301
```

---

## GRPCRoute with ALPN

`GRPCRoute` is a first-class Gateway API resource.  Cilium's Envoy proxy
handles HTTP/2 with ALPN negotiation (`h2`) natively.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-route
  namespace: default
spec:
  parentRefs:
    - name: main-gateway
      sectionName: https
  hostnames:
    - grpc.example.com
  rules:
    - matches:
        - method:
            service: mypackage.MyService
            method: GetItem
      backendRefs:
        - name: grpc-svc
          port: 50051
    - matches:
        - method:
            service: mypackage.MyService
      backendRefs:
        - name: grpc-svc
          port: 50051
```

The HTTPS listener on the Gateway terminates TLS.  Because Cilium's Envoy
advertises `h2` in its ALPN list, gRPC clients negotiate HTTP/2 automatically.
No extra annotations or protocol hints are needed.

> **Backend health:** For gRPC health checking, deploy a gRPC health probe
> sidecar or use Cilium's L7 health checking when available.

---

## TLSRoute (Passthrough)

Use `TLSRoute` when the backend terminates TLS itself (e.g., a service that
must see the original client certificate).

First, add a passthrough listener to the Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: passthrough-gateway
  namespace: default
spec:
  gatewayClassName: cilium
  listeners:
    - name: tls-passthrough
      protocol: TLS
      port: 443
      tls:
        mode: Passthrough
      allowedRoutes:
        namespaces:
          from: All
```

Then create the TLSRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: backend-tls-passthrough
  namespace: default
spec:
  parentRefs:
    - name: passthrough-gateway
      sectionName: tls-passthrough
  hostnames:
    - secure.example.com
  rules:
    - backendRefs:
        - name: secure-backend
          port: 8443
```

---

## Comparison: Gateway API vs Ingress

| Aspect | Ingress (v1) | Gateway API |
|---|---|---|
| API maturity | Stable, frozen feature set | GA (v1), actively evolving |
| Role separation | Single resource, one owner | GatewayClass (infra) / Gateway (platform) / Route (app dev) |
| Protocol scope | HTTP/HTTPS only | HTTP, gRPC, TLS, TCP, UDP |
| Header matching | Annotation-dependent | Native `matches.headers` |
| Traffic splitting | Annotation-dependent (canary) | Native `backendRefs.weight` |
| Cross-namespace | Not supported | `ReferenceGrant` CRD |
| TLS passthrough | Annotation-dependent | `TLSRoute` with `mode: Passthrough` |
| Multiple controllers | `ingressClassName` | `gatewayClassName` + `parentRefs` |

**When to use Gateway API with Cilium:**

- You already run Cilium as your CNI and want to avoid extra data-plane pods.
- You need gRPC-native routing, traffic splitting, or header-based routing
  without vendor annotations.
- You want clear role separation (platform team manages Gateway, app teams
  manage Routes).

**When Ingress may still be preferable:**

- Existing Ingress manifests across many repos -- migration cost is high.
- You need features only available via NGINX annotations (e.g., ModSecurity WAF,
  Lua snippets).
