# Network Policy on Talos

Network policies control pod-to-pod and pod-to-external traffic at L3/L4 (and L7 with Cilium). Without network policies, all pods can communicate with all other pods across all namespaces. This guide covers default-deny baselines, namespace isolation, allow-list patterns, and CNI-specific policies.

**Prerequisite**: Your CNI must support NetworkPolicy. Flannel (Talos default) does **not**. Use Cilium or Calico. See [deploy/networking](../../deploy/networking/README.md).

---

## 1. Default-Deny Policies

Every namespace should start with a default-deny policy. This ensures that traffic is blocked unless explicitly allowed.

### Deny All Ingress and Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

This blocks all inbound and outbound traffic for every pod in the `production` namespace.

### Deny All Ingress Only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### Deny All Egress Only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### Apply Default-Deny to All Namespaces

```bash
# Apply default-deny to every non-system namespace
for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n' | grep -v '^kube-'); do
  kubectl apply -n "$ns" -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
EOF
done
```

---

## 2. Allow DNS Egress

After applying default-deny-egress, pods cannot resolve DNS. Allow DNS as the first egress rule:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

---

## 3. Namespace Isolation Patterns

### Allow Traffic Within Same Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - podSelector: {}
```

### Allow Ingress from a Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress-controller
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
```

### Allow Ingress from Monitoring Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9090
        - protocol: TCP
          port: 8080
```

---

## 4. Allow-List Patterns

### Frontend to Backend

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

### Backend to Database

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-allow-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgresql
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432
```

### Allow Egress to External API

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    # DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
    # External API endpoint
    - to:
        - ipBlock:
            cidr: 203.0.113.0/24
      ports:
        - protocol: TCP
          port: 443
```

---

## 5. CiliumNetworkPolicy (L7)

Cilium extends Kubernetes NetworkPolicy with L7 (HTTP, gRPC, Kafka, DNS) visibility and enforcement.

### L7 HTTP Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-l7-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
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
              - method: POST
                path: "/api/v1/orders"
              - method: GET
                path: "/healthz"
```

This allows the frontend to call `GET /api/v1/*`, `POST /api/v1/orders`, and `GET /healthz` on the backend. All other HTTP methods and paths are denied.

### L7 DNS Policy

Control which external domains pods can resolve:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: restrict-dns-egress
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
    # Allow DNS queries
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*.production.svc.cluster.local"
              - matchName: "api.stripe.com"
              - matchName: "api.sendgrid.com"
    # Allow HTTPS to resolved IPs
    - toFQDNs:
        - matchName: "api.stripe.com"
        - matchName: "api.sendgrid.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

### L7 gRPC Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: grpc-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: grpc-server
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: grpc-client
      toPorts:
        - ports:
            - port: "50051"
              protocol: TCP
          rules:
            http:
              - method: POST
                path: "/mypackage.MyService/.*"
                headers:
                  - "content-type: application/grpc"
```

### Cilium Cluster-Wide Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: deny-external-by-default
spec:
  endpointSelector: {}
  egress:
    # Allow cluster-internal traffic
    - toEntities:
        - cluster
    # Allow DNS
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
```

---

## 6. Calico GlobalNetworkPolicy

If using Calico instead of Cilium, use GlobalNetworkPolicy for cluster-wide rules.

### Default Deny (Calico)

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny
spec:
  selector: all()
  types:
    - Ingress
    - Egress
```

### Allow DNS (Calico)

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-dns
spec:
  selector: all()
  types:
    - Egress
  egress:
    - action: Allow
      protocol: UDP
      destination:
        selector: k8s-app == 'kube-dns'
        ports:
          - 53
    - action: Allow
      protocol: TCP
      destination:
        selector: k8s-app == 'kube-dns'
        ports:
          - 53
  order: 100
```

### Allow In-Namespace Traffic (Calico)

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  selector: all()
  types:
    - Ingress
    - Egress
  ingress:
    - action: Allow
      source:
        selector: all()
  egress:
    - action: Allow
      destination:
        selector: all()
```

---

## 7. Monitoring Network Policies

### Verify Policies Are Applied

```bash
# List all network policies
kubectl get networkpolicy -A

# Describe a specific policy
kubectl describe networkpolicy default-deny-all -n production

# List Cilium policies (if using Cilium)
kubectl get ciliumnetworkpolicy -A
kubectl get ciliumclusterwidenetworkpolicy
```

### Cilium Policy Verdict Monitoring

```bash
# Watch policy verdicts in real-time (requires Cilium CLI)
cilium monitor --type policy-verdict

# Check endpoint policy status
cilium endpoint list

# Hubble flow logs (if Hubble is enabled)
hubble observe --namespace production --verdict DROPPED
hubble observe --namespace production --verdict FORWARDED

# Hubble UI for visual traffic flow
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

### Test Connectivity

```bash
# Test from one pod to another
kubectl exec -n production deploy/frontend -- wget -qO- --timeout=3 http://backend:8080/healthz

# Test DNS resolution
kubectl exec -n production deploy/frontend -- nslookup backend.production.svc.cluster.local

# Cilium connectivity test (comprehensive)
cilium connectivity test
```

---

## 8. Complete Namespace Security Template

Combine default-deny, DNS allow, same-namespace allow, and ingress-controller allow into a single set:

```yaml
# 1. Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# 2. Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
---
# 3. Allow same-namespace traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - podSelector: {}
---
# 4. Allow ingress controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 443
---
# 5. Allow Prometheus scraping
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9090
        - protocol: TCP
          port: 8080
```

---

## 9. Network Policy Checklist for Talos Clusters

1. Verify your CNI supports NetworkPolicy (Cilium or Calico, not Flannel)
2. Apply default-deny to every non-system namespace
3. Always allow DNS egress to `kube-system` as the first egress rule
4. Use `podSelector` labels for service-to-service allow rules
5. Use `namespaceSelector` for cross-namespace access (ingress controller, monitoring)
6. Use `ipBlock` for external endpoints; prefer CiliumNetworkPolicy `toFQDNs` when using Cilium
7. Use Cilium L7 policies for HTTP path and method filtering in security-sensitive applications
8. Monitor dropped traffic with Hubble (Cilium) or Calico flow logs
9. Test policies with `kubectl exec` connectivity checks before and after applying
10. Combine with RBAC namespace isolation (see [rbac.md](./rbac.md)) for defense in depth
