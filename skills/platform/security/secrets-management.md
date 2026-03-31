# Secrets Management on Talos

Talos encrypts etcd data at rest with Secretbox by default, but Kubernetes Secrets are still base64-encoded in manifests and accessible to anyone with RBAC read access. Production clusters need a secrets management solution that keeps secret material out of Git and provides rotation, auditing, and access control.

---

## 1. Sealed Secrets

Sealed Secrets encrypts secrets client-side using a public key. Only the Sealed Secrets controller (running in-cluster) can decrypt them. Encrypted SealedSecret resources are safe to commit to Git.

### Install Sealed Secrets Controller

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller
```

Verify:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key
```

### Install kubeseal CLI

```bash
# macOS
brew install kubeseal

# Linux
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | grep tag_name | cut -d '"' -f4)
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION#v}-linux-amd64.tar.gz"
tar -xzf kubeseal-*.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

### Create a SealedSecret

```bash
# Create a regular Secret manifest (do not apply it)
kubectl create secret generic db-credentials \
  --namespace production \
  --from-literal=username=app_user \
  --from-literal=password='S3cur3P@ssw0rd!' \
  --dry-run=client -o yaml > db-credentials-secret.yaml

# Seal it using the cluster's public key
kubeseal --format yaml < db-credentials-secret.yaml > db-credentials-sealed.yaml

# Delete the plaintext secret
rm db-credentials-secret.yaml

# Apply the sealed secret (safe to commit to Git)
kubectl apply -f db-credentials-sealed.yaml
```

### SealedSecret YAML

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    username: AgBy3i... # encrypted by kubeseal
    password: AgCtr8... # encrypted by kubeseal
  template:
    metadata:
      name: db-credentials
      namespace: production
    type: Opaque
```

The controller decrypts this into a standard Kubernetes Secret in the `production` namespace.

### Scoping Modes

| Scope | Flag | Behavior |
|---|---|---|
| `strict` (default) | `--scope strict` | Bound to exact name + namespace. Cannot be renamed or moved. |
| `namespace-wide` | `--scope namespace-wide` | Bound to namespace. Can be renamed within that namespace. |
| `cluster-wide` | `--scope cluster-wide` | Can be used in any namespace with any name. |

```bash
# Seal with namespace-wide scope
kubeseal --format yaml --scope namespace-wide < secret.yaml > sealed.yaml
```

### Key Rotation and Backup

```bash
# Backup the sealing key (store securely outside the cluster)
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key-backup.yaml

# The controller generates a new key every 30 days by default
# Old keys are retained for decryption of existing SealedSecrets
# Re-encrypt existing SealedSecrets with the latest key:
kubeseal --re-encrypt < existing-sealed.yaml > re-encrypted-sealed.yaml
```

---

## 2. External Secrets Operator (ESO)

ESO synchronizes secrets from external providers (Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault) into Kubernetes Secrets. The secret material never lives in Git.

### Install ESO via Helm

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true \
  --set webhook.port=9443 \
  --set certController.requeueInterval=5m
```

Verify:

```bash
kubectl get pods -n external-secrets
kubectl get crd | grep external-secrets
```

### SecretStore for HashiCorp Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-store
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "production-role"
          serviceAccountRef:
            name: vault-auth-sa
```

### SecretStore for AWS Secrets Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-store
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key
```

For IRSA (IAM Roles for Service Accounts) on EKS or equivalent:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-store-irsa
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-service-account
```

### SecretStore for GCP Secret Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-store
  namespace: production
spec:
  provider:
    gcpsm:
      projectID: my-gcp-project
      auth:
        secretRef:
          secretAccessKeySecretRef:
            name: gcp-credentials
            key: service-account-key
```

### ClusterSecretStore (Cluster-Wide)

Use `ClusterSecretStore` when multiple namespaces need access to the same provider:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-cluster-store
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "cluster-wide-role"
          serviceAccountRef:
            name: vault-auth-sa
            namespace: external-secrets
```

### ExternalSecret — Sync a Secret from Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-store
    kind: SecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@db.production.svc:5432/appdb"
  data:
    - secretKey: username
      remoteRef:
        key: production/database
        property: username
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

### ExternalSecret — Sync from AWS Secrets Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-keys
  namespace: production
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: aws-store
    kind: SecretStore
  target:
    name: api-keys
    creationPolicy: Owner
  data:
    - secretKey: stripe-key
      remoteRef:
        key: production/stripe
        property: api_key
    - secretKey: sendgrid-key
      remoteRef:
        key: production/sendgrid
        property: api_key
```

### Verify ESO Sync

```bash
# Check ExternalSecret status
kubectl get externalsecret -n production
kubectl describe externalsecret db-credentials -n production

# The status should show:
# Status: SecretSynced
# Conditions: Ready=True

# Verify the Kubernetes Secret was created
kubectl get secret db-credentials -n production
```

---

## 3. HashiCorp Vault on Kubernetes

Run Vault as a first-class Kubernetes workload with the Agent Injector for automatic sidecar injection or the CSI provider for volume-mounted secrets.

### Install Vault via Helm

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.ha.enabled=true \
  --set server.ha.replicas=3 \
  --set server.ha.raft.enabled=true \
  --set server.dataStorage.size=10Gi \
  --set server.dataStorage.storageClass=ceph-block \
  --set injector.enabled=true \
  --set csi.enabled=true
```

### Initialize and Unseal Vault

```bash
# Initialize Vault (one time)
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > vault-init.json

# Store vault-init.json securely — it contains unseal keys and root token

# Unseal each replica (repeat for vault-1, vault-2)
UNSEAL_KEY_1=$(jq -r '.unseal_keys_b64[0]' vault-init.json)
UNSEAL_KEY_2=$(jq -r '.unseal_keys_b64[1]' vault-init.json)
UNSEAL_KEY_3=$(jq -r '.unseal_keys_b64[2]' vault-init.json)

kubectl exec -n vault vault-0 -- vault operator unseal "$UNSEAL_KEY_1"
kubectl exec -n vault vault-0 -- vault operator unseal "$UNSEAL_KEY_2"
kubectl exec -n vault vault-0 -- vault operator unseal "$UNSEAL_KEY_3"

# Join raft cluster (vault-1 and vault-2)
kubectl exec -n vault vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
kubectl exec -n vault vault-2 -- vault operator raft join http://vault-0.vault-internal:8200
```

### Configure Kubernetes Auth Method

```bash
ROOT_TOKEN=$(jq -r '.root_token' vault-init.json)

kubectl exec -n vault vault-0 -- sh -c "
  export VAULT_TOKEN='${ROOT_TOKEN}'
  vault auth enable kubernetes
  vault write auth/kubernetes/config \
    kubernetes_host='https://kubernetes.default.svc:443'
  vault secrets enable -path=secret kv-v2
"
```

### Create a Policy and Role

```bash
kubectl exec -n vault vault-0 -- sh -c "
  export VAULT_TOKEN='${ROOT_TOKEN}'

  # Create a policy
  vault policy write production-read - <<POLICY
  path \"secret/data/production/*\" {
    capabilities = [\"read\"]
  }
POLICY

  # Create a Kubernetes auth role
  vault write auth/kubernetes/role/production-app \
    bound_service_account_names=app-sa \
    bound_service_account_namespaces=production \
    policies=production-read \
    ttl=1h
"
```

### Agent Injector — Sidecar Pattern

The Vault Agent Injector automatically injects a sidecar container that fetches secrets and writes them to a shared volume.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "production-app"
        vault.hashicorp.com/agent-inject-secret-db-creds: "secret/data/production/database"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "secret/data/production/database" -}}
          export DATABASE_URL="postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@db:5432/app"
          {{- end -}}
    spec:
      serviceAccountName: app-sa
      containers:
        - name: myapp
          image: myapp:latest
          command: ["/bin/sh", "-c", "source /vault/secrets/db-creds && exec /app"]
          resources:
            limits:
              memory: 256Mi
              cpu: 250m
            requests:
              memory: 128Mi
              cpu: 100m
```

### Vault CSI Provider

Mount secrets as files via the CSI driver (no sidecar needed):

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db-creds
  namespace: production
spec:
  provider: vault
  parameters:
    vaultAddress: "http://vault.vault.svc:8200"
    roleName: "production-app"
    objects: |
      - objectName: "db-username"
        secretPath: "secret/data/production/database"
        secretKey: "username"
      - objectName: "db-password"
        secretPath: "secret/data/production/database"
        secretKey: "password"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: app-sa
      containers:
        - name: myapp
          image: myapp:latest
          volumeMounts:
            - name: secrets
              mountPath: /mnt/secrets
              readOnly: true
          resources:
            limits:
              memory: 256Mi
              cpu: 250m
            requests:
              memory: 128Mi
              cpu: 100m
      volumes:
        - name: secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: vault-db-creds
```

---

## 4. Comparison Table

| Feature | Sealed Secrets | External Secrets Operator | HashiCorp Vault |
|---|---|---|---|
| Secret storage | In-cluster (encrypted CRD) | External provider | Vault server (in-cluster or external) |
| GitOps friendly | Yes (SealedSecret in Git) | Yes (ExternalSecret in Git) | Partial (annotations in Git, secrets in Vault) |
| Rotation | Manual re-seal | Automatic via `refreshInterval` | Automatic (dynamic secrets, leases) |
| Multi-provider | No | Yes (Vault, AWS SM, GCP SM, Azure KV) | Vault only (but Vault has many backends) |
| Complexity | Low | Medium | High |
| Dependencies | Sealed Secrets controller | ESO + external provider | Vault cluster + storage + unseal mechanism |
| Audit trail | No | Provider-dependent | Yes (built-in audit log) |
| Dynamic secrets | No | No | Yes (database, AWS, PKI) |
| Cost | Free | Free + provider costs | Free (OSS) or paid (Enterprise) |

### Decision Guide

- **Small team, GitOps-first, no external provider** → **Sealed Secrets**
- **Multi-cloud or existing cloud secret stores** → **External Secrets Operator**
- **Enterprise requirements: dynamic secrets, rotation, audit** → **HashiCorp Vault**
- **Vault + GitOps** → **ESO with Vault SecretStore** (best of both: GitOps manifests + Vault as backend)

---

## 5. Talos-Specific Considerations

- **Secretbox encryption**: Talos already encrypts etcd at rest. This protects Kubernetes Secrets on disk but not in transit via the API.
- **No host path for Vault storage**: Vault's Raft storage needs a PersistentVolume. Use a CSI-provisioned PVC (Ceph, Longhorn) — do not attempt host-path storage on Talos.
- **ServiceAccount tokens**: Kubernetes 1.24+ uses bound, time-limited tokens. Vault's Kubernetes auth works with these by default.
- **Network policies**: Restrict which namespaces can reach the Vault service. See [network-policy.md](./network-policy.md).
