# NGINX Ingress Controller

## Install with Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --version 4.12.0 \
  --set controller.kind=DaemonSet \
  --set controller.service.type=LoadBalancer \
  --set controller.allowSnippetAnnotations=false \
  --set controller.config.use-forwarded-headers="true" \
  --set controller.config.compute-full-forwarded-for="true" \
  --set controller.config.proxy-body-size="50m" \
  --set controller.metrics.enabled=true \
  --set controller.metrics.serviceMonitor.enabled=true \
  --wait
```

> Use `DaemonSet` on bare-metal Talos clusters so every node can receive
> traffic.  Switch to `Deployment` with HPA if you prefer elastic scaling.

Verify:

```bash
kubectl -n ingress-nginx get pods
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

The `EXTERNAL-IP` column should show an IP from your MetalLB pool.

---

## IngressClass

The Helm chart creates an `IngressClass` named `nginx` and marks it as the
default.  To make it explicit or run multiple controllers:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

---

## Example Ingress with TLS (cert-manager)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-example-com-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-svc
                port:
                  number: 80
```

---

## Common Annotations

### Redirects and Rewrites

```yaml
# Force HTTPS
nginx.ingress.kubernetes.io/ssl-redirect: "true"

# Rewrite target -- strip /api prefix before forwarding
nginx.ingress.kubernetes.io/rewrite-target: /$2
# with path: /api(/|$)(.*)

# Permanent redirect
nginx.ingress.kubernetes.io/permanent-redirect: "https://new.example.com"
```

### Rate Limiting

```yaml
nginx.ingress.kubernetes.io/limit-rps: "20"
nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
nginx.ingress.kubernetes.io/limit-connections: "10"
```

### Timeouts and Body Size

```yaml
nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
nginx.ingress.kubernetes.io/proxy-body-size: "100m"
```

### Authentication

```yaml
# Basic auth from a secret
nginx.ingress.kubernetes.io/auth-type: basic
nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret
nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

# External OAuth2 proxy
nginx.ingress.kubernetes.io/auth-url: "https://oauth2.example.com/oauth2/auth"
nginx.ingress.kubernetes.io/auth-signin: "https://oauth2.example.com/oauth2/start?rd=$scheme://$host$escaped_request_uri"
```

### CORS

```yaml
nginx.ingress.kubernetes.io/enable-cors: "true"
nginx.ingress.kubernetes.io/cors-allow-origin: "https://frontend.example.com"
nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type"
nginx.ingress.kubernetes.io/cors-max-age: "3600"
```

### Backend Protocol

```yaml
# Talk gRPC to the backend
nginx.ingress.kubernetes.io/backend-protocol: "GRPC"

# Talk HTTPS to the backend
nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
```

---

## TCP and UDP Services

NGINX Ingress can proxy arbitrary TCP/UDP ports via ConfigMaps.

### 1. Create the ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  "5432": "databases/postgresql:5432"       # namespace/service:port
  "6379": "cache/redis-master:6379"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: udp-services
  namespace: ingress-nginx
data:
  "53": "kube-system/coredns:53"
```

### 2. Tell the controller about the ConfigMaps

Add these Helm values (or patch the existing install):

```bash
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --reuse-values \
  --set tcp.5432="databases/postgresql:5432" \
  --set tcp.6379="cache/redis-master:6379" \
  --set udp.53="kube-system/coredns:53"
```

### 3. Expose the ports on the Service

The Helm chart automatically patches the `LoadBalancer` service to include the
declared TCP/UDP ports when the `--set tcp.*` and `--set udp.*` values are used.

Verify:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller -o yaml | grep -A3 "5432"
```

---

## Health Checks and Debugging

```bash
# Check controller logs
kubectl -n ingress-nginx logs -l app.kubernetes.io/name=ingress-nginx --tail=100

# Dump the generated nginx.conf
kubectl -n ingress-nginx exec deploy/ingress-nginx-controller -- cat /etc/nginx/nginx.conf

# Test config validity
kubectl -n ingress-nginx exec deploy/ingress-nginx-controller -- nginx -t

# View active backends
kubectl -n ingress-nginx exec deploy/ingress-nginx-controller -- \
  curl -s http://localhost:10246/configuration/backends | jq '.[].name'
```
