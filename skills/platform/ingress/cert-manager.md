# cert-manager

## Install with Helm

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.1 \
  --set crds.enabled=true \
  --set prometheus.enabled=true \
  --set prometheus.servicemonitor.enabled=true \
  --wait
```

Verify:

```bash
kubectl -n cert-manager get pods
kubectl get crd certificates.cert-manager.io clusterissuers.cert-manager.io
```

All three pods (`cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook`)
must be Running before creating issuers.

---

## ClusterIssuer: Let's Encrypt with HTTP-01

HTTP-01 requires the ingress controller to serve the ACME challenge token on
port 80.  This is the simplest setup for publicly reachable services.

### Staging (for testing -- avoids rate limits)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: platform-team@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

### Production

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform-team@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

Usage on an Ingress:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: app-example-com-tls
```

cert-manager will detect the annotation, create an `Order`, solve the HTTP-01
challenge, and store the signed certificate in `app-example-com-tls`.

---

## ClusterIssuer: Let's Encrypt with DNS-01

DNS-01 is required for wildcard certificates and works even when the cluster is
not reachable from the internet.

### Example: Cloudflare

```bash
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token='YOUR_CLOUDFLARE_API_TOKEN'
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform-team@example.com
    privateKeySecretRef:
      name: letsencrypt-dns-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
        selector:
          dnsZones:
            - example.com
```

### Example: AWS Route 53

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-route53
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform-team@example.com
    privateKeySecretRef:
      name: letsencrypt-route53-account-key
    solvers:
      - dns01:
          route53:
            region: us-east-1
            hostedZoneID: Z0123456789ABCDEFGHIJ
        selector:
          dnsZones:
            - example.com
```

> For Route 53, cert-manager uses IRSA (IAM Roles for Service Accounts) or
> ambient credentials.  On bare-metal Talos you will typically create an IAM user
> and store credentials in a Secret, or use `ambient: true` with instance
> metadata if running on EC2.

### Wildcard Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-example-com
  namespace: default
spec:
  secretName: wildcard-example-com-tls
  issuerRef:
    name: letsencrypt-dns
    kind: ClusterIssuer
  commonName: "*.example.com"
  dnsNames:
    - "*.example.com"
    - example.com
```

---

## Certificate Resource

For cases where you need explicit control (not annotation-driven):

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-cert
  namespace: default
spec:
  secretName: api-example-com-tls
  duration: 2160h        # 90 days
  renewBefore: 720h      # renew 30 days before expiry
  isCA: false
  privateKey:
    algorithm: ECDSA
    size: 256
  usages:
    - server auth
    - client auth
  dnsNames:
    - api.example.com
    - api-internal.example.com
  ipAddresses:
    - 10.0.50.100
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

Check certificate status:

```bash
kubectl get certificate -A
kubectl describe certificate api-cert -n default
kubectl get certificaterequest -A
kubectl get order -A
kubectl get challenge -A
```

---

## Self-Signed CA

For development, air-gapped, or internal-only environments.

### Step 1: Self-signed issuer to bootstrap the CA

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-bootstrap
spec:
  selfSigned: {}
```

### Step 2: Create the CA certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: internal-ca
  secretName: internal-ca-key-pair
  duration: 87600h       # 10 years
  renewBefore: 8760h     # 1 year before expiry
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-bootstrap
    kind: ClusterIssuer
```

### Step 3: ClusterIssuer backed by the CA

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: internal-ca-key-pair
```

Now any Certificate referencing `internal-ca-issuer` will be signed by the
internal CA:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-app-cert
  namespace: default
spec:
  secretName: internal-app-tls
  issuerRef:
    name: internal-ca-issuer
    kind: ClusterIssuer
  dnsNames:
    - app.internal.example.com
```

---

## trust-manager

trust-manager distributes CA bundles so that workloads trust certificates signed
by the internal CA (or any other CA).

### Install

```bash
helm install trust-manager jetstack/trust-manager \
  --namespace cert-manager \
  --version v0.13.0 \
  --set app.trust.namespace=cert-manager \
  --wait
```

### Create a Bundle

A `Bundle` resource collects CA certificates from multiple sources and projects
them into a ConfigMap or Secret in every namespace (or selected namespaces).

```yaml
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: cluster-trust-bundle
spec:
  sources:
    # Include the default Debian/Alpine CA bundle
    - useDefaultCAs: true
    # Include our internal CA
    - secret:
        name: internal-ca-key-pair
        key: ca.crt
    # Include an additional CA from a ConfigMap
    - configMap:
        name: extra-ca
        key: ca-bundle.crt
  target:
    configMap:
      key: ca-certificates.crt
    namespaceSelector:
      matchLabels: {}     # all namespaces
```

### Mount in Workloads

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: default
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          env:
            - name: SSL_CERT_FILE
              value: /etc/ssl/trust/ca-certificates.crt
          volumeMounts:
            - name: trust-bundle
              mountPath: /etc/ssl/trust
              readOnly: true
      volumes:
        - name: trust-bundle
          configMap:
            name: cluster-trust-bundle
```

---

## Troubleshooting

```bash
# Check cert-manager controller logs
kubectl -n cert-manager logs deploy/cert-manager --tail=200

# Check webhook logs (TLS handshake errors)
kubectl -n cert-manager logs deploy/cert-manager-webhook --tail=100

# Describe a stuck certificate
kubectl describe certificate <name> -n <namespace>

# Inspect the current challenge
kubectl get challenge -A
kubectl describe challenge <name> -n <namespace>

# Manually trigger renewal (delete the secret)
kubectl delete secret <tls-secret-name> -n <namespace>
# cert-manager will re-issue automatically

# Verify a certificate's dates
kubectl get secret <tls-secret-name> -n <namespace> -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -dates -subject -issuer
```
