# Pod Security on Talos

Pod Security Standards (PSS) define three levels of security restriction for pods. Talos clusters should enforce at least the `baseline` profile cluster-wide and `restricted` in production namespaces. This guide covers the built-in PodSecurity admission controller and policy engines (Kyverno, OPA Gatekeeper) for fine-grained enforcement.

---

## 1. Pod Security Standards (PSS)

Kubernetes defines three profiles:

| Profile | Description | Use Case |
|---|---|---|
| `privileged` | Unrestricted. No security constraints applied. | System namespaces (`kube-system`), CNI, storage drivers |
| `baseline` | Prevents known privilege escalations. Blocks `hostNetwork`, `hostPID`, `privileged` containers. | General workloads, default cluster policy |
| `restricted` | Maximum restriction. Requires non-root, drops all capabilities, read-only root FS, `RuntimeDefault` seccomp. | Production application namespaces |

### What Each Profile Blocks

**Baseline** blocks:
- `hostNetwork`, `hostPID`, `hostIPC`
- `privileged: true`
- Capabilities beyond the allowed set (`NET_BIND_SERVICE` only)
- `hostPath` volumes
- Container ports below 1024 (non-root)

**Restricted** adds:
- Must run as non-root (`runAsNonRoot: true`)
- Must drop `ALL` capabilities
- Must set `allowPrivilegeEscalation: false`
- Must use `RuntimeDefault` or `Localhost` seccomp profile
- Must not use `hostPath` volumes

---

## 2. PodSecurity Admission Controller

The PodSecurity admission controller is built into Kubernetes (v1.25+, stable). It enforces PSS profiles at the namespace level using labels.

### Namespace Labels

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Reject pods that violate the restricted profile
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    # Log violations against restricted profile
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    # Warn users about violations
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

### Baseline for General Namespaces

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

This enforces `baseline` (rejects violations) and warns about `restricted` violations (does not reject, shows warning to user).

### Exempt System Namespaces

System namespaces that run CNI, storage drivers, or monitoring agents need `privileged`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: latest
```

### Apply Labels to Existing Namespaces

```bash
# Enforce restricted on production namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest

# Enforce baseline cluster-wide on default namespace
kubectl label namespace default \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest

# Dry-run to check impact before enforcing
kubectl label --dry-run=server --overwrite namespace staging \
  pod-security.kubernetes.io/enforce=restricted
```

### Cluster-Wide Defaults via Admission Configuration

To set default PSS levels for all namespaces (including new ones), configure the admission controller in the Talos machine config:

```yaml
cluster:
  apiServer:
    admissionControl:
      - name: PodSecurity
        configuration:
          apiVersion: pod-security.admission.config.k8s.io/v1
          kind: PodSecurityConfiguration
          defaults:
            enforce: baseline
            enforce-version: latest
            audit: restricted
            audit-version: latest
            warn: restricted
            warn-version: latest
          exemptions:
            namespaces:
              - kube-system
              - kube-public
              - kube-node-lease
              - cilium
              - rook-ceph
              - longhorn-system
```

This sets `baseline` enforcement as the cluster default while warning about `restricted` violations. System namespaces are exempted.

---

## 3. Kyverno Policy Engine

Kyverno provides Kubernetes-native policy management using CustomResourceDefinitions. It can validate, mutate, and generate resources. Use it when you need more granular policies than PSS labels provide.

### Install Kyverno via Helm

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --set replicaCount=3 \
  --set admissionController.replicas=3 \
  --set backgroundController.replicas=2 \
  --set cleanupController.replicas=2 \
  --set reportsController.replicas=2
```

Verify installation:

```bash
kubectl get pods -n kyverno
kubectl get clusterpolicy
```

### ClusterPolicy: Require Labels

Require all Deployments to have `app.kubernetes.io/name` and `app.kubernetes.io/owner` labels:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
  annotations:
    policies.kyverno.io/title: Require Labels
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: require-app-labels
      match:
        any:
          - resources:
              kinds:
                - Deployment
                - StatefulSet
                - DaemonSet
      validate:
        message: >-
          The label `app.kubernetes.io/name` is required.
          The label `app.kubernetes.io/owner` is required.
        pattern:
          metadata:
            labels:
              app.kubernetes.io/name: "?*"
              app.kubernetes.io/owner: "?*"
```

### ClusterPolicy: Disallow Privileged Containers

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
  annotations:
    policies.kyverno.io/title: Disallow Privileged Containers
    policies.kyverno.io/category: Pod Security
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: deny-privileged
      match:
        any:
          - resources:
              kinds:
                - Pod
      exclude:
        any:
          - resources:
              namespaces:
                - kube-system
                - cilium
                - rook-ceph
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            =(initContainers):
              - =(securityContext):
                  =(privileged): false
            containers:
              - =(securityContext):
                  =(privileged): false
```

### ClusterPolicy: Require Resource Limits

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
  annotations:
    policies.kyverno.io/title: Require Resource Limits
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: require-limits
      match:
        any:
          - resources:
              kinds:
                - Pod
      exclude:
        any:
          - resources:
              namespaces:
                - kube-system
                - kyverno
      validate:
        message: "CPU and memory limits are required for all containers."
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

### ClusterPolicy: Mutate — Add Default Security Context

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-security-context
spec:
  rules:
    - name: add-run-as-non-root
      match:
        any:
          - resources:
              kinds:
                - Pod
      exclude:
        any:
          - resources:
              namespaces:
                - kube-system
      mutate:
        patchStrategicMerge:
          spec:
            securityContext:
              runAsNonRoot: true
              seccompProfile:
                type: RuntimeDefault
```

### Audit Mode vs. Enforce Mode

Set `validationFailureAction` to control behavior:

- `Enforce` — reject resources that violate the policy
- `Audit` — allow resources but report violations in the policy report

```bash
# Check policy reports
kubectl get policyreport -A
kubectl get clusterpolicyreport

# Describe a specific report for violation details
kubectl describe policyreport -n production
```

---

## 4. OPA Gatekeeper Alternative

OPA Gatekeeper uses Rego policies via ConstraintTemplates. Use Gatekeeper if your organization already uses OPA or needs Rego's expressiveness.

### Install Gatekeeper via Helm

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update

helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace \
  --set replicas=3 \
  --set audit.replicas=1 \
  --set audit.logLevel=INFO
```

### ConstraintTemplate: Required Labels

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
```

### Constraint: Apply the Template

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-labels
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
  parameters:
    labels:
      - "app.kubernetes.io/name"
      - "app.kubernetes.io/owner"
```

### Gatekeeper Audit

```bash
# Check constraint violations
kubectl get k8srequiredlabels require-app-labels -o yaml

# The status.violations field lists all non-compliant resources
```

---

## 5. Kyverno vs. Gatekeeper Comparison

| Feature | Kyverno | OPA Gatekeeper |
|---|---|---|
| Policy language | Kubernetes-native YAML | Rego (OPA) |
| Learning curve | Low | Medium–High |
| Mutation support | Yes (native) | Yes (v3.7+, external data) |
| Generation support | Yes (create resources) | No |
| Policy reports | Built-in (PolicyReport CRD) | Audit via status field |
| Community policies | kyverno.io/policies | gatekeeper-library |
| Resource overhead | Medium | Medium |

**Recommendation**: Use **Kyverno** for most Talos clusters unless you already have OPA/Rego expertise. Kyverno policies are Kubernetes-native YAML and easier to maintain.

---

## 6. Pod Security Checklist for Talos Clusters

1. Set cluster-wide `baseline` enforcement via admission configuration in machine config
2. Label production namespaces with `restricted` enforcement
3. Exempt system namespaces (`kube-system`, CNI namespace, storage namespace)
4. Deploy Kyverno or Gatekeeper for policies beyond what PSS covers (labels, resource limits, image registries)
5. Run Kyverno in `Audit` mode first, review reports, then switch to `Enforce`
6. Validate with dry-run before applying enforcement changes

```bash
# Test a pod against the restricted profile without applying
kubectl run test-pod --image=nginx --dry-run=server -o yaml 2>&1
```
