# RBAC on Talos Kubernetes

Kubernetes Role-Based Access Control (RBAC) governs who can do what within the cluster. Talos clusters start with a powerful admin kubeconfig generated at bootstrap. Production clusters must scope access down using least-privilege Roles, namespace-isolated ServiceAccounts, and audit logging.

---

## 1. RBAC Model

Kubernetes RBAC has four key objects:

| Object | Scope | Purpose |
|---|---|---|
| `Role` | Namespace | Grants permissions within a single namespace |
| `ClusterRole` | Cluster | Grants permissions cluster-wide or across all namespaces |
| `RoleBinding` | Namespace | Binds a Role or ClusterRole to subjects within a namespace |
| `ClusterRoleBinding` | Cluster | Binds a ClusterRole to subjects cluster-wide |

Subjects can be: `User`, `Group`, or `ServiceAccount`.

---

## 2. Role and ClusterRole

### Namespace-Scoped Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-developer
  namespace: production
rules:
  # Read pods, logs, events
  - apiGroups: [""]
    resources: ["pods", "pods/log", "events", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  # Manage deployments and replicasets
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Read secrets (but not create/delete)
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
```

### ClusterRole for Read-Only Cluster Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
  - apiGroups: [""]
    resources: ["namespaces", "nodes", "persistentvolumes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["networkpolicies", "ingresses"]
    verbs: ["get", "list", "watch"]
```

### ClusterRole for Namespace Admin

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-admin
rules:
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources: ["*"]
    verbs: ["*"]
  # Explicitly deny cluster-scoped resources
  # (bind this with a RoleBinding, not ClusterRoleBinding)
```

---

## 3. RoleBinding and ClusterRoleBinding

### Bind Role to a User in a Namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: production
subjects:
  - kind: User
    name: jane.doe@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: app-developer
  apiGroup: rbac.authorization.k8s.io
```

### Bind ClusterRole to a Group Cluster-Wide

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sre-cluster-viewer
subjects:
  - kind: Group
    name: sre-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-viewer
  apiGroup: rbac.authorization.k8s.io
```

### Bind ClusterRole Scoped to a Single Namespace

Use a `RoleBinding` that references a `ClusterRole` — grants the ClusterRole permissions only within the binding's namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: namespace-admin-binding
  namespace: production
subjects:
  - kind: User
    name: team-lead@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io
```

### Bind to a ServiceAccount

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: cicd-deployer
    namespace: cicd
roleRef:
  kind: Role
  name: app-developer
  apiGroup: rbac.authorization.k8s.io
```

---

## 4. ServiceAccount Best Practices

### Create Dedicated ServiceAccounts

Never use the `default` ServiceAccount for workloads. Create purpose-specific accounts:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
automountServiceAccountToken: false
```

### Disable Auto-Mount Where Not Needed

Kubernetes 1.24+ issues bound, time-limited tokens instead of long-lived ones. Still, disable auto-mount for pods that do not call the Kubernetes API:

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
    spec:
      serviceAccountName: myapp-sa
      automountServiceAccountToken: false
      containers:
        - name: myapp
          image: myapp:latest
          resources:
            limits:
              memory: 256Mi
              cpu: 250m
            requests:
              memory: 128Mi
              cpu: 100m
```

### Scoped Token for API Access

When a pod does need API access, create a projected token with a specific audience and expiration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-consumer
  namespace: production
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false
  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/tokens
          readOnly: true
      resources:
        limits:
          memory: 128Mi
          cpu: 100m
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              path: api-token
              expirationSeconds: 3600
              audience: "https://kubernetes.default.svc"
```

---

## 5. Least-Privilege Patterns

### Pattern 1: Developer Access (Read + Limited Write)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/portforward", "services", "configmaps", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["deployments/scale"]
    verbs: ["update", "patch"]
```

### Pattern 2: CI/CD Deployer (Write Deployments Only)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
```

### Pattern 3: Monitoring (Read Everything, Write Nothing)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "nodes", "services", "endpoints", "namespaces"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list"]
```

### Pattern 4: Namespace Self-Service

Allow a team to manage resources within their namespace but not create new namespaces or access other namespaces:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-admin
  namespace: team-alpha
subjects:
  - kind: Group
    name: team-alpha
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin  # built-in ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

The built-in `admin` ClusterRole grants full control within a namespace (excluding resource quotas and the namespace itself) when bound with a `RoleBinding`.

---

## 6. Namespace Isolation

Combine RBAC with other controls for full namespace isolation:

```yaml
# 1. Create the namespace with PSS labels
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    team: alpha
---
# 2. Resource quota to prevent resource abuse
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "10"
---
# 3. LimitRange to set defaults and caps per pod
apiVersion: v1
kind: LimitRange
metadata:
  name: team-alpha-limits
  namespace: team-alpha
spec:
  limits:
    - type: Container
      default:
        cpu: 250m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "2"
        memory: 4Gi
---
# 4. NetworkPolicy to isolate the namespace (see network-policy.md)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

---

## 7. Audit Logging

Enable the Kubernetes audit log to track who accessed what. This is critical for compliance and incident response.

### Audit Policy

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all changes to secrets at Metadata level (do not log secret data)
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  # Log all changes to RBAC objects at RequestResponse level
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  # Log pod exec and port-forward
  - level: Request
    resources:
      - group: ""
        resources: ["pods/exec", "pods/portforward"]
  # Log all write operations at Request level
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
  # Log read operations at Metadata level
  - level: Metadata
    verbs: ["get", "list", "watch"]
  # Ignore health checks and system events
  - level: None
    users: ["system:kube-proxy"]
  - level: None
    resources:
      - group: ""
        resources: ["endpoints", "events"]
    verbs: ["get", "list", "watch"]
```

### Enable Audit Logging in Talos Machine Config

```yaml
cluster:
  apiServer:
    extraArgs:
      audit-policy-file: /etc/kubernetes/audit-policy.yaml
      audit-log-path: /var/log/kubernetes/audit.log
      audit-log-maxage: "30"
      audit-log-maxbackup: "10"
      audit-log-maxsize: "100"
    extraVolumes:
      - name: audit-policy
        hostPath: /var/etc/kubernetes/audit-policy.yaml
        mountPath: /etc/kubernetes/audit-policy.yaml
        readOnly: true
      - name: audit-log
        hostPath: /var/log/kubernetes
        mountPath: /var/log/kubernetes
```

Write the audit policy to the Talos node:

```bash
# Upload audit policy to the node
talosctl cp audit-policy.yaml /var/etc/kubernetes/audit-policy.yaml --nodes 10.0.0.21
```

### Query Audit Logs

```bash
# Read audit logs from the control plane node
talosctl read /var/log/kubernetes/audit.log --nodes 10.0.0.21 | tail -50

# Filter for secret access
talosctl read /var/log/kubernetes/audit.log --nodes 10.0.0.21 | grep '"resource":"secrets"'

# Filter for RBAC changes
talosctl read /var/log/kubernetes/audit.log --nodes 10.0.0.21 | grep '"resource":"clusterroles"'
```

---

## 8. Checking Effective Permissions

```bash
# Check what a user can do in a namespace
kubectl auth can-i --list --as jane.doe@example.com -n production

# Check a specific permission
kubectl auth can-i create deployments --as jane.doe@example.com -n production

# Check what a ServiceAccount can do
kubectl auth can-i --list --as system:serviceaccount:production:myapp-sa -n production

# Check cluster-wide permissions
kubectl auth can-i --list --as system:serviceaccount:cicd:deployer

# Find all RoleBindings for a user
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.subjects[]? | .name == "jane.doe@example.com") | {kind: .kind, name: .metadata.name, namespace: .metadata.namespace}'
```

---

## 9. RBAC Checklist for Talos Clusters

1. Generate a separate kubeconfig for each human user or CI/CD system (do not share the admin kubeconfig)
2. Create namespace-scoped Roles for application teams
3. Use RoleBindings (not ClusterRoleBindings) to scope ClusterRoles to specific namespaces
4. Create dedicated ServiceAccounts for each workload with `automountServiceAccountToken: false`
5. Use the built-in `view`, `edit`, and `admin` ClusterRoles where possible
6. Enable audit logging on all control plane nodes
7. Review permissions quarterly: `kubectl auth can-i --list --as <user> -n <namespace>`
8. Combine RBAC with ResourceQuotas, LimitRanges, NetworkPolicies, and PSS for full namespace isolation
