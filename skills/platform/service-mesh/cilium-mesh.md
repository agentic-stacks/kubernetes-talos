# Cilium Service Mesh on Talos

Cilium provides service mesh capabilities natively through eBPF without requiring sidecar proxies. If your cluster already runs Cilium as its CNI, enabling mesh features adds zero extra pods per workload.

Reference: https://docs.cilium.io/en/latest/network/servicemesh/

## Prerequisites

Cilium must be installed as the cluster CNI. See [Cilium CNI guide](../../deploy/networking/cilium.md) for installation on Talos.

Minimum Cilium version: **1.14+** for mutual authentication, **1.15+** for CiliumEnvoyConfig GA.

## Enabling Mesh Features

### Mutual Authentication (mTLS)

Cilium mutual authentication uses SPIFFE identities to establish mTLS between services at the eBPF level. No sidecars are injected.

Enable during Cilium install or upgrade:

```bash
cilium install \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445 \
    --set authentication.mutual.spire.enabled=true \
    --set authentication.mutual.spire.install.enabled=true
```

Or upgrade an existing installation:

```bash
cilium upgrade \
    --set authentication.mutual.spire.enabled=true \
    --set authentication.mutual.spire.install.enabled=true
```

This deploys a SPIRE server and agent alongside Cilium. The SPIRE agent runs as a DaemonSet and issues SVID certificates to workloads.

### Verify mutual authentication

```bash
# Check SPIRE agent is running
kubectl get pods -n cilium-spire

# Check Cilium agents report mutual auth enabled
cilium status | grep -i mutual

# Check identity assignments
kubectl exec -n kube-system ds/cilium -- cilium identity list
```

### Require authentication on a policy

Once mutual authentication is enabled, enforce it per-service using CiliumNetworkPolicy:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: require-mutual-auth
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend-api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      authentication:
        mode: required
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
```

The `authentication.mode: required` field tells Cilium to reject any connection that has not completed mutual authentication. Traffic from `frontend` to `backend-api` on port 8080 must present a valid SPIFFE identity.

## L7 Network Policies

CiliumNetworkPolicy supports L7 rules for HTTP, Kafka, and DNS without sidecars. Cilium uses a per-node Envoy proxy (deployed automatically) for L7 inspection.

### HTTP-aware policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-get-only
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend-api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: "/api/v1/.*"
              - method: GET
                path: "/healthz"
```

This allows only GET requests to `/api/v1/*` and `/healthz` from `frontend` to `backend-api`. All other HTTP methods and paths are denied.

### Kafka-aware policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: kafka-produce-only
  namespace: streaming
spec:
  endpointSelector:
    matchLabels:
      app: kafka-broker
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: event-producer
      toPorts:
        - ports:
            - port: "9092"
              protocol: TCP
          rules:
            kafka:
              - apiKey: produce
                topic: "events-.*"
```

### DNS-aware policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-external-dns
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend-api
  egress:
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
          rules:
            dns:
              - matchPattern: "*.example.com"
    - toFQDNs:
        - matchPattern: "*.example.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

## CiliumEnvoyConfig for L7 Routing

CiliumEnvoyConfig (CEC) provides advanced L7 traffic management — header-based routing, traffic splitting, retries, and timeouts — by configuring the per-node Envoy proxy directly.

### Traffic splitting (canary deployment)

```yaml
apiVersion: cilium.io/v2
kind: CiliumEnvoyConfig
metadata:
  name: backend-api-canary
  namespace: production
spec:
  services:
    - name: backend-api
      namespace: production
  backendServices:
    - name: backend-api-stable
      namespace: production
    - name: backend-api-canary
      namespace: production
  resources:
    - "@type": type.googleapis.com/envoy.config.listener.v3.Listener
      name: backend-api-listener
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: backend-api
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend-api
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            weighted_clusters:
                              clusters:
                                - name: "production/backend-api-stable"
                                  weight: 90
                                - name: "production/backend-api-canary"
                                  weight: 10
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

### Header-based routing

```yaml
apiVersion: cilium.io/v2
kind: CiliumEnvoyConfig
metadata:
  name: backend-api-header-routing
  namespace: production
spec:
  services:
    - name: backend-api
      namespace: production
  backendServices:
    - name: backend-api-v1
      namespace: production
    - name: backend-api-v2
      namespace: production
  resources:
    - "@type": type.googleapis.com/envoy.config.listener.v3.Listener
      name: backend-api-listener
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: backend-api
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend-api
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/"
                            headers:
                              - name: x-api-version
                                exact_match: "v2"
                          route:
                            cluster: "production/backend-api-v2"
                        - match:
                            prefix: "/"
                          route:
                            cluster: "production/backend-api-v1"
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

### Retry and timeout policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumEnvoyConfig
metadata:
  name: backend-api-resilience
  namespace: production
spec:
  services:
    - name: backend-api
      namespace: production
  backendServices:
    - name: backend-api
      namespace: production
  resources:
    - "@type": type.googleapis.com/envoy.config.listener.v3.Listener
      name: backend-api-listener
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: backend-api
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend-api
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            cluster: "production/backend-api"
                            timeout: 15s
                            retry_policy:
                              retry_on: "5xx,reset,connect-failure"
                              num_retries: 3
                              per_try_timeout: 5s
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

## Hubble Observability

Hubble provides network flow visibility, DNS monitoring, and HTTP/Kafka protocol-level observability. It is built into Cilium.

### Enable Hubble

```bash
cilium hubble enable --ui
```

Or set during install:

```bash
--set hubble.enabled=true \
--set hubble.relay.enabled=true \
--set hubble.ui.enabled=true \
--set hubble.metrics.enableOpenMetrics=true \
--set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"
```

### Observe flows

```bash
# Port-forward Hubble Relay
cilium hubble port-forward &

# Observe all flows
hubble observe

# Filter by namespace
hubble observe --namespace production

# Filter by HTTP status
hubble observe --protocol http --http-status 500

# Filter by verdict (dropped traffic)
hubble observe --verdict DROPPED

# Export flows as JSON
hubble observe --namespace production -o json > flows.json
```

### Access Hubble UI

```bash
cilium hubble ui
```

This opens a browser with the Hubble UI showing a service map with live traffic flows, DNS queries, and HTTP request/response details.

## Verification

```bash
# Check Cilium status including mesh features
cilium status --verbose

# Verify mutual authentication is active
cilium encrypt status

# Run connectivity tests (includes L7 policy tests)
kubectl create namespace cilium-test
kubectl label namespace cilium-test pod-security.kubernetes.io/enforce=privileged
cilium connectivity test --namespace cilium-test

# Clean up test namespace
kubectl delete namespace cilium-test
```
