# Istio Service Mesh on Talos

Istio is a feature-rich service mesh built on Envoy proxies. On Talos, the Istio CNI plugin is required because the immutable OS does not allow `istio-init` containers to manipulate iptables from within pods.

Reference: https://istio.io/latest/docs/setup/platform-setup/

## Prerequisites

- Kubernetes cluster running on Talos Linux
- `kubectl` configured with cluster access
- `istioctl` CLI installed

```bash
# Install istioctl
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.24.2 sh -
export PATH=$PWD/istio-1.24.2/bin:$PATH

# Verify
istioctl version --remote=false
```

## Talos Kernel Considerations

Talos ships a minimal, locked-down kernel. Istio requires awareness of:

| Concern | Impact | Mitigation |
|---|---|---|
| No iptables from pods | `istio-init` init container fails | Use Istio CNI plugin (`components.cni.enabled=true`) |
| No kernel module loading | Envoy cannot load modules | Not an issue — Envoy runs in userspace |
| Read-only filesystem | Envoy needs writable dirs | Istio uses `emptyDir` volumes by default — no changes needed |
| cgroup management | Talos manages cgroups | No Istio-specific impact |

## Installation

### Install with istioctl and CNI plugin

The `demo` profile is shown for completeness. For production, use the `default` profile and customize.

```bash
istioctl install --set profile=default \
    --set components.cni.enabled=true \
    --set values.cni.cniBinDir=/opt/cni/bin \
    --set values.cni.cniConfDir=/etc/cni/net.d \
    --set values.cni.chained=true \
    --set values.cni.excludeNamespaces[0]=kube-system \
    --set values.cni.excludeNamespaces[1]=istio-system \
    --set meshConfig.accessLogFile=/dev/stdout \
    --set meshConfig.defaultConfig.holdApplicationUntilProxyStarts=true \
    -y
```

Key flags explained:

- `components.cni.enabled=true` — installs the Istio CNI DaemonSet instead of using `istio-init` containers. This is **required on Talos**.
- `values.cni.chained=true` — chains Istio CNI after the primary CNI (Cilium, Flannel, Calico).
- `holdApplicationUntilProxyStarts=true` — prevents application containers from starting before the Envoy sidecar is ready.

### Install with Helm

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

# Install base (CRDs)
helm install istio-base istio/base \
    --namespace istio-system \
    --create-namespace

# Install istiod (control plane)
helm install istiod istio/istiod \
    --namespace istio-system \
    --set meshConfig.accessLogFile=/dev/stdout \
    --set meshConfig.defaultConfig.holdApplicationUntilProxyStarts=true

# Install CNI plugin (required on Talos)
helm install istio-cni istio/cni \
    --namespace istio-system \
    --set cni.cniBinDir=/opt/cni/bin \
    --set cni.cniConfDir=/etc/cni/net.d \
    --set cni.chained=true \
    --set cni.excludeNamespaces[0]=kube-system \
    --set cni.excludeNamespaces[1]=istio-system
```

### Verify installation

```bash
istioctl verify-install
kubectl get pods -n istio-system
kubectl get crd | grep istio
```

## Sidecar Injection

Istio injects Envoy sidecars into pods via a mutating admission webhook. Enable injection per namespace:

```bash
kubectl label namespace production istio-injection=enabled
```

Verify injection is enabled:

```bash
kubectl get namespace production --show-labels | grep istio
```

To exclude a specific pod from injection:

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/inject: "false"
```

Restart existing deployments to inject sidecars:

```bash
kubectl rollout restart deployment -n production
```

## Gateway and VirtualService

### Gateway (ingress entry point)

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: app-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: app-tls-cert
      hosts:
        - "app.example.com"
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "app.example.com"
      tls:
        httpsRedirect: true
```

### VirtualService (routing rules)

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: backend-api
  namespace: production
spec:
  hosts:
    - backend-api
    - "app.example.com"
  gateways:
    - app-gateway
    - mesh
  http:
    - match:
        - uri:
            prefix: /api/v2
          headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: backend-api
            subset: canary
            port:
              number: 8080
    - match:
        - uri:
            prefix: /api
      route:
        - destination:
            host: backend-api
            subset: stable
            port:
              number: 8080
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: 5xx,reset,connect-failure
      fault:
        delay:
          percentage:
            value: 0.1
          fixedDelay: 5s
```

### DestinationRule (load balancing and subsets)

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: backend-api
  namespace: production
spec:
  host: backend-api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: stable
      labels:
        version: v1
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
    - name: canary
      labels:
        version: v2
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
```

## PeerAuthentication (mTLS)

### Mesh-wide STRICT mTLS

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

This enforces mTLS for all services across the mesh. Any service without a sidecar will be unable to communicate with meshed services.

### Namespace-level PERMISSIVE (migration mode)

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: PERMISSIVE
```

PERMISSIVE accepts both plaintext and mTLS. Use this during migration when some services do not yet have sidecars.

### Per-port exception

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: backend-api
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend-api
  mtls:
    mode: STRICT
  portLevelMtls:
    9090:
      mode: PERMISSIVE
```

This keeps STRICT mTLS on all ports except 9090 (e.g., a Prometheus metrics port scraped by a non-meshed collector).

## Traffic Splitting (Canary Deployments)

Use VirtualService weights to split traffic between subsets defined in a DestinationRule:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: backend-api
  namespace: production
spec:
  hosts:
    - backend-api
  http:
    - route:
        - destination:
            host: backend-api
            subset: stable
          weight: 90
        - destination:
            host: backend-api
            subset: canary
          weight: 10
```

Progressive rollout — shift weight gradually:

```bash
# 10% canary
kubectl apply -f vs-canary-10.yaml

# 50% canary
kubectl apply -f vs-canary-50.yaml

# 100% canary (promote)
kubectl apply -f vs-canary-100.yaml
```

## Authorization Policy

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: backend-api-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend-api
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/frontend
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
    - from:
        - source:
            principals:
              - cluster.local/ns/monitoring/sa/prometheus
      to:
        - operation:
            methods: ["GET"]
            paths: ["/metrics"]
```

## Install Ingress Gateway

On Talos, use a dedicated Istio ingress gateway deployment:

```bash
# With istioctl
istioctl install --set profile=default \
    --set components.cni.enabled=true \
    --set values.cni.cniBinDir=/opt/cni/bin \
    --set values.cni.cniConfDir=/etc/cni/net.d \
    --set values.cni.chained=true \
    --set components.ingressGateways[0].name=istio-ingressgateway \
    --set components.ingressGateways[0].enabled=true \
    -y

# Or with Helm
helm install istio-ingress istio/gateway \
    --namespace istio-ingress \
    --create-namespace
```

## Observability Add-Ons

```bash
# Install Kiali, Prometheus, Grafana, Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/jaeger.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/kiali.yaml

# Access Kiali dashboard
istioctl dashboard kiali
```

## Troubleshooting on Talos

```bash
# Check sidecar injection status
istioctl analyze --namespace production

# Verify proxy status for a pod
istioctl proxy-status
istioctl proxy-config cluster deployment/backend-api -n production

# Check CNI plugin is running
kubectl get pods -n istio-system -l k8s-app=istio-cni-node

# View Envoy access logs
kubectl logs deployment/backend-api -n production -c istio-proxy

# Debug mTLS issues
istioctl authn tls-check backend-api.production.svc.cluster.local
```

## Uninstall

```bash
# Remove with istioctl
istioctl uninstall --purge -y
kubectl delete namespace istio-system

# Remove with Helm
helm uninstall istio-ingress -n istio-ingress
helm uninstall istio-cni -n istio-system
helm uninstall istiod -n istio-system
helm uninstall istio-base -n istio-system
kubectl delete namespace istio-system istio-ingress
```
