# Linkerd Service Mesh on Talos

Linkerd is a lightweight service mesh built on Rust-based micro-proxies. It provides mTLS by default, golden metrics (success rate, latency, throughput), and traffic management with minimal resource overhead (~10 MB memory per sidecar).

Reference: https://linkerd.io/2/getting-started/

## Prerequisites

- Kubernetes cluster running on Talos Linux
- `kubectl` configured with cluster access
- `linkerd` CLI installed

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$HOME/.linkerd2/bin:$PATH

# Verify CLI
linkerd version --client
```

## Talos Considerations

| Concern | Impact | Mitigation |
|---|---|---|
| No iptables from pods | `linkerd-init` init container fails | Use `--set cniEnabled=true` to install Linkerd CNI plugin |
| Read-only filesystem | Proxy needs writable tmpfs | Linkerd uses `emptyDir` by default — no changes needed |
| No kernel module loading | Not an issue | Linkerd proxy runs entirely in userspace |

## Installation

### Step 1: Pre-flight checks

```bash
linkerd check --pre
```

This validates cluster prerequisites. Expect warnings about missing CRDs (they do not exist yet).

### Step 2: Install CRDs

```bash
linkerd install --crds | kubectl apply -f -
```

### Step 3: Install Linkerd CNI plugin (required on Talos)

The CNI plugin configures iptables at the node level instead of using init containers, which is required on Talos.

```bash
linkerd install-cni | kubectl apply -f -
```

Or with Helm:

```bash
helm repo add linkerd-edge https://helm.linkerd.io/edge
helm repo update

helm install linkerd-cni linkerd-edge/linkerd2-cni \
    --namespace linkerd-cni \
    --create-namespace
```

Wait for CNI plugin to be ready:

```bash
kubectl rollout status daemonset/linkerd-cni -n linkerd-cni
```

### Step 4: Install Linkerd control plane

```bash
linkerd install --set cniEnabled=true | kubectl apply -f -
```

Or with Helm:

```bash
helm install linkerd-control-plane linkerd-edge/linkerd-control-plane \
    --namespace linkerd \
    --create-namespace \
    --set cniEnabled=true \
    --set identity.issuer.scheme=kubernetes.io/tls
```

### Step 5: Verify installation

```bash
linkerd check
```

All checks should pass. If the CNI check fails, verify the CNI DaemonSet is running:

```bash
kubectl get pods -n linkerd-cni
kubectl get pods -n linkerd
```

## Sidecar Injection

Linkerd uses annotations to control proxy injection. Annotate a namespace to inject all pods:

```bash
kubectl annotate namespace production linkerd.io/inject=enabled
```

Or annotate individual deployments:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: production
spec:
  template:
    metadata:
      annotations:
        linkerd.io/inject: enabled
```

Restart existing deployments to inject:

```bash
kubectl rollout restart deployment -n production
```

Verify injection:

```bash
linkerd check --proxy --namespace production
kubectl get pods -n production -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
```

Each pod should have a `linkerd-proxy` container.

### Exclude a pod from injection

```yaml
metadata:
  annotations:
    linkerd.io/inject: disabled
```

## mTLS by Default

Linkerd enables mutual TLS automatically for all meshed traffic. No configuration is required. When two pods both have the Linkerd proxy injected, all traffic between them is encrypted and authenticated.

### Verify mTLS

```bash
# Check mTLS status for a namespace
linkerd viz stat deploy -n production

# The SECURED column shows the percentage of mTLS-encrypted traffic
# 100% means all traffic is encrypted

# Detailed edge view showing TLS status per connection
linkerd viz edges deploy -n production
```

### Trust anchor rotation

Linkerd uses a trust anchor certificate for the mesh identity system. Rotate it before expiry:

```bash
# Check certificate expiry
linkerd check --proxy

# Generate new trust anchor
step certificate create root.linkerd.cluster.local ca.crt ca.key \
    --profile root-ca --no-password --insecure

# Update trust anchor
linkerd upgrade --identity-trust-anchors-file ca.crt | kubectl apply -f -
```

## Traffic Splitting with TrafficSplit

TrafficSplit is the SMI (Service Mesh Interface) resource for weighted routing:

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: backend-api
  namespace: production
spec:
  service: backend-api
  backends:
    - service: backend-api-stable
      weight: 900
    - service: backend-api-canary
      weight: 100
```

This sends 90% of traffic to `backend-api-stable` and 10% to `backend-api-canary`. Both backend services must select pods with matching labels.

Supporting services:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  namespace: production
spec:
  selector:
    app: backend-api
  ports:
    - port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend-api-stable
  namespace: production
spec:
  selector:
    app: backend-api
    version: v1
  ports:
    - port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend-api-canary
  namespace: production
spec:
  selector:
    app: backend-api
    version: v2
  ports:
    - port: 8080
```

## HTTPRoute (Gateway API)

Linkerd 2.14+ supports Gateway API HTTPRoute for traffic management:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: backend-api
  namespace: production
spec:
  parentRefs:
    - name: backend-api
      kind: Service
      group: core
      port: 8080
  rules:
    - matches:
        - headers:
            - name: x-canary
              value: "true"
      backendRefs:
        - name: backend-api-canary
          port: 8080
    - backendRefs:
        - name: backend-api-stable
          port: 8080
          weight: 90
        - name: backend-api-canary
          port: 8080
          weight: 10
```

This routes requests with `x-canary: true` header entirely to canary, while splitting remaining traffic 90/10.

## ServiceProfile

ServiceProfile defines per-route metrics and retry behavior:

```yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: backend-api.production.svc.cluster.local
  namespace: production
spec:
  routes:
    - name: GET /api/v1/users
      condition:
        method: GET
        pathRegex: /api/v1/users
      responseClasses:
        - condition:
            status:
              min: 500
              max: 599
          isFailure: true
      isRetryable: true
      timeout: 10s
    - name: POST /api/v1/users
      condition:
        method: POST
        pathRegex: /api/v1/users
      isRetryable: false
      timeout: 30s
    - name: GET /healthz
      condition:
        method: GET
        pathRegex: /healthz
      timeout: 5s
  retryBudget:
    retryRatio: 0.2
    minRetriesPerSecond: 10
    ttl: 10s
```

Generate a ServiceProfile from an OpenAPI spec:

```bash
linkerd profile --open-api swagger.json backend-api -n production | kubectl apply -f -
```

Or from live traffic:

```bash
linkerd profile --tap deploy/backend-api -n production --tap-duration 60s | kubectl apply -f -
```

## Linkerd Viz Dashboard

Linkerd Viz provides a web dashboard with golden metrics (success rate, request rate, latency percentiles) and live topology.

### Install Viz extension

```bash
linkerd viz install | kubectl apply -f -

# Wait for Viz to be ready
linkerd viz check
```

### Access the dashboard

```bash
linkerd viz dashboard &
```

This opens a browser with the Linkerd dashboard. The dashboard shows:

- **Top-level overview**: namespace-level success rates and traffic volume
- **Deployment detail**: per-deployment golden metrics with live sparklines
- **Live traffic**: real-time request flows between services
- **Routes**: per-route metrics when ServiceProfiles are configured

### CLI equivalents

```bash
# Namespace overview
linkerd viz stat deploy -n production

# Per-route metrics (requires ServiceProfile)
linkerd viz routes deploy/backend-api -n production

# Top (live request stream)
linkerd viz top deploy/backend-api -n production

# Tap (live request/response inspection)
linkerd viz tap deploy/backend-api -n production

# Edges (service-to-service connections with mTLS status)
linkerd viz edges deploy -n production
```

## Multi-Cluster

Linkerd multi-cluster links two or more clusters, allowing services in one cluster to discover and route to services in another, with mTLS maintained across cluster boundaries.

### Prerequisites

- Each cluster has Linkerd installed with a shared trust anchor
- Clusters can reach each other's gateway (LoadBalancer or NodePort)

### Install multi-cluster extension on each cluster

```bash
linkerd multicluster install | kubectl apply -f -
linkerd multicluster check
```

### Link clusters

From the cluster that should discover services in the remote cluster:

```bash
# On the remote cluster, generate a link credential
linkerd multicluster link --cluster-name remote-cluster \
    --api-server-address https://remote-api-server:6443 | kubectl apply -f -
```

### Export a service for cross-cluster access

On the remote cluster, mark a service as exported:

```bash
kubectl label service backend-api -n production mirror.linkerd.io/exported=true
```

The local cluster will now see `backend-api-remote-cluster` as a mirrored service. Traffic to this service is routed to the remote cluster's gateway with full mTLS.

### Verify multi-cluster

```bash
# Check link status
linkerd multicluster check

# See mirrored services
kubectl get services -n production | grep remote-cluster

# View cross-cluster traffic metrics
linkerd viz stat deploy -n production --from deploy/frontend
```

## Uninstall

```bash
# Remove Viz
linkerd viz uninstall | kubectl delete -f -

# Remove multi-cluster (if installed)
linkerd multicluster uninstall | kubectl delete -f -

# Remove control plane
linkerd uninstall | kubectl delete -f -

# Remove CNI plugin
linkerd install-cni --uninstall | kubectl delete -f -
# or with Helm
helm uninstall linkerd-cni -n linkerd-cni
```
