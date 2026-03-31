# Traefik Ingress Controller

## Install with Helm

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --version 33.2.1 \
  --set deployment.kind=DaemonSet \
  --set service.type=LoadBalancer \
  --set ports.web.redirectTo.port=websecure \
  --set ports.websecure.tls.enabled=true \
  --set providers.kubernetesIngress.enabled=true \
  --set providers.kubernetesCRD.enabled=true \
  --set providers.kubernetesCRD.allowCrossNamespace=true \
  --set metrics.prometheus.enabled=true \
  --set metrics.prometheus.serviceMonitor.enabled=true \
  --set logs.general.level=INFO \
  --set logs.access.enabled=true \
  --wait
```

Verify:

```bash
kubectl -n traefik get pods
kubectl -n traefik get svc traefik
```

---

## IngressRoute CRD

Traefik's native CRD provides features beyond the standard Ingress resource.

### Basic IngressRoute with TLS

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: app-ingress
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.example.com`)
      kind: Rule
      services:
        - name: app-svc
          port: 80
    - match: Host(`app.example.com`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: api-svc
          port: 8080
      middlewares:
        - name: rate-limit
        - name: api-strip-prefix
  tls:
    secretName: app-example-com-tls
```

### IngressRoute with cert-manager

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: app-auto-tls
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.example.com`)
      kind: Rule
      services:
        - name: app-svc
          port: 80
  tls:
    secretName: app-example-com-tls
    domains:
      - main: app.example.com
```

### Weighted Traffic Splitting

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: canary-route
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.example.com`)
      kind: Rule
      services:
        - name: app-v1
          port: 80
          weight: 90
        - name: app-v2
          port: 80
          weight: 10
  tls:
    secretName: app-example-com-tls
```

---

## Middleware Examples

### Rate Limiting

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: default
spec:
  rateLimit:
    average: 50           # requests per second
    burst: 100
    period: 1s
    sourceCriterion:
      ipStrategy:
        depth: 1          # use first X-Forwarded-For hop
```

### Basic Authentication

```bash
# Generate htpasswd secret
htpasswd -nBb admin 'S3cur3P@ss' | kubectl create secret generic basic-auth \
  --namespace default --from-file=users=/dev/stdin
```

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: default
spec:
  basicAuth:
    secret: basic-auth
    realm: "Restricted"
    removeHeader: true
```

### Forward Authentication (OAuth2 Proxy / Authelia)

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: forward-auth
  namespace: default
spec:
  forwardAuth:
    address: http://authelia.auth.svc.cluster.local:9091/api/verify?rd=https://auth.example.com
    trustForwardHeader: true
    authResponseHeaders:
      - Remote-User
      - Remote-Groups
      - Remote-Email
```

### HTTPS Redirect

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: default
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

### Strip Path Prefix

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: api-strip-prefix
  namespace: default
spec:
  stripPrefix:
    prefixes:
      - /api
    forceSlash: false
```

### Headers (Security)

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: security-headers
  namespace: default
spec:
  headers:
    frameDeny: true
    contentTypeNosniff: true
    browserXssFilter: true
    stsIncludeSubdomains: true
    stsPreload: true
    stsSeconds: 31536000
    customFrameOptionsValue: "SAMEORIGIN"
    contentSecurityPolicy: "default-src 'self'"
    referrerPolicy: "strict-origin-when-cross-origin"
```

### Chaining Multiple Middlewares

Reference middlewares in order on the IngressRoute:

```yaml
spec:
  routes:
    - match: Host(`app.example.com`)
      kind: Rule
      middlewares:
        - name: redirect-https
        - name: security-headers
        - name: rate-limit
        - name: forward-auth
      services:
        - name: app-svc
          port: 80
```

---

## TCP and UDP IngressRoute

### TCP (e.g., PostgreSQL)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRouteTCP
metadata:
  name: postgres-tcp
  namespace: databases
spec:
  entryPoints:
    - postgres       # must be defined in Traefik static config
  routes:
    - match: HostSNI(`*`)
      services:
        - name: postgresql
          port: 5432
```

Add the custom entrypoint via Helm values:

```bash
helm upgrade traefik traefik/traefik \
  --namespace traefik \
  --reuse-values \
  --set "ports.postgres.port=5432" \
  --set "ports.postgres.expose.default=true" \
  --set "ports.postgres.protocol=TCP"
```

### UDP (e.g., DNS)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRouteUDP
metadata:
  name: dns-udp
  namespace: kube-system
spec:
  entryPoints:
    - dns-udp
  routes:
    - services:
        - name: coredns
          port: 53
```

---

## Dashboard Access

The Traefik dashboard is disabled by default in production installs.  Expose it
securely via an IngressRoute with authentication:

```bash
helm upgrade traefik traefik/traefik \
  --namespace traefik \
  --reuse-values \
  --set api.dashboard=true \
  --set api.insecure=false
```

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.example.com`)
      kind: Rule
      middlewares:
        - name: basic-auth
          namespace: default
        - name: security-headers
          namespace: default
      services:
        - name: api@internal
          kind: TraefikService
  tls:
    secretName: traefik-dashboard-tls
```

> **Never** set `api.insecure=true` outside of local development.  Always protect
> the dashboard behind authentication middleware.

---

## Debugging

```bash
# Controller logs
kubectl -n traefik logs -l app.kubernetes.io/name=traefik --tail=100

# List all routers (requires API enabled)
kubectl -n traefik port-forward svc/traefik 9000:9000 &
curl -s http://localhost:9000/api/http/routers | jq '.[].name'

# Check middleware chain for a router
curl -s http://localhost:9000/api/http/routers | jq '.[] | select(.name | contains("app")) | .middlewares'
```
