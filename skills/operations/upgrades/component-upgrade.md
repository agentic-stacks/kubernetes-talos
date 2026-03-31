# Component Upgrade

Upgrading platform components deployed via Helm — CNI (Cilium, Calico), CSI (Rook-Ceph, Longhorn), ingress controllers, observability stacks, and other Helm-managed workloads.

These upgrades are entirely independent from Talos OS and Kubernetes version upgrades.

---

## General Helm Upgrade Workflow

### 1. Check Current Version

```bash
# List all Helm releases across all namespaces
helm list --all-namespaces

# Check a specific release
helm list -n kube-system -f cilium
helm history cilium -n kube-system
```

### 2. Review Upstream Release Notes

Before upgrading any component, read its changelog:

- **Cilium:** https://docs.cilium.io/en/stable/operations/upgrade/
- **Calico:** https://docs.tigera.io/calico/latest/operations/upgrading
- **Rook-Ceph:** https://rook.io/docs/rook/latest/Upgrade/rook-ceph-upgrade/
- **Longhorn:** https://longhorn.io/docs/latest/deploy/upgrade/
- **ingress-nginx:** https://github.com/kubernetes/ingress-nginx/releases

**Pay attention to:**
- Breaking changes in values.yaml schema
- Required CRD updates (Helm does not upgrade CRDs by default)
- Minimum Kubernetes version requirements
- Migration steps that must be run manually

### 3. Update Helm Repository

```bash
helm repo update
```

### 4. Check Available Versions

```bash
# List available versions of a chart
helm search repo cilium/cilium --versions | head -20
```

### 5. Diff Before Applying

```bash
# Install the helm-diff plugin if not already present
helm plugin install https://github.com/databus23/helm-diff 2>/dev/null || true

# Preview changes
helm diff upgrade cilium cilium/cilium \
  --version 1.16.5 \
  --namespace kube-system \
  --values values/cilium.yaml
```

### 6. Test in Staging First

Always upgrade components in a staging/dev cluster before production. Verify:
- Pods come up healthy
- Network connectivity works (for CNI upgrades)
- Storage provisioning works (for CSI upgrades)
- Ingress routing works (for ingress controller upgrades)
- Monitoring/alerting works (for observability upgrades)

---

## CNI Upgrade (Cilium Example)

```bash
# Check current version
kubectl exec -n kube-system ds/cilium -- cilium version
helm list -n kube-system -f cilium

# Update CRDs first (Helm does not manage CRD lifecycle on upgrade)
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.16.5/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumnetworkpolicies.yaml
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.16.5/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumclusterwidenetworkpolicies.yaml
# Apply all CRDs from the target version — check Cilium docs for complete list

# Upgrade
helm upgrade cilium cilium/cilium \
  --version 1.16.5 \
  --namespace kube-system \
  --values values/cilium.yaml \
  --wait --timeout 10m

# Verify
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium
kubectl exec -n kube-system ds/cilium -- cilium status
cilium connectivity test 2>/dev/null || echo "Run 'cilium connectivity test' if cilium CLI is installed"
```

**CNI upgrade risks:** A botched CNI upgrade can take down all pod networking. Always:
- Have out-of-band access to nodes (Talos API on port 50000 does not depend on CNI)
- Be prepared to rollback immediately

---

## CSI Upgrade (Rook-Ceph Example)

```bash
# Check current version
helm list -n rook-ceph

# Review Rook upgrade guide — Rook has strict upgrade ordering:
# 1. Upgrade rook-ceph operator
# 2. Wait for operator to reconcile
# 3. Upgrade ceph cluster (if separate chart)

# Upgrade operator
helm upgrade rook-ceph rook-release/rook-ceph \
  --version 1.15.2 \
  --namespace rook-ceph \
  --values values/rook-ceph.yaml \
  --wait --timeout 15m

# Monitor operator reconciliation
kubectl -n rook-ceph get pods -w
kubectl -n rook-ceph logs deploy/rook-ceph-operator -f

# Verify Ceph health after upgrade
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd status
```

---

## Ingress Controller Upgrade (ingress-nginx Example)

```bash
# Check current version
helm list -n ingress-nginx

# Diff the upgrade
helm diff upgrade ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.11.3 \
  --namespace ingress-nginx \
  --values values/ingress-nginx.yaml

# Upgrade
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.11.3 \
  --namespace ingress-nginx \
  --values values/ingress-nginx.yaml \
  --wait --timeout 5m

# Verify — check that ingress traffic still works
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
curl -I https://your-app.example.com
```

---

## Rollback

If an upgrade causes issues, rollback to the previous revision:

```bash
# Check revision history
helm history cilium -n kube-system

# Rollback to previous revision
helm rollback cilium -n kube-system

# Rollback to a specific revision number
helm rollback cilium 3 -n kube-system

# Verify
helm list -n kube-system -f cilium
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium
```

**Rollback caveats:**
- Helm rollback reverts the Helm release (templates/values) but does **not** revert CRD changes
- If you upgraded CRDs before the Helm upgrade, you may need to manually revert them
- Some operators perform data migrations on upgrade that are not reversible — check component docs

---

## Version Pinning

Always pin component versions in your values files and Helm commands. Never use `latest` or omit `--version`.

```bash
# Good — pinned version
helm upgrade cilium cilium/cilium --version 1.16.5 -n kube-system -f values/cilium.yaml

# Bad — unpinned, gets whatever is newest
helm upgrade cilium cilium/cilium -n kube-system -f values/cilium.yaml
```

Maintain a version manifest file for your cluster:

```yaml
# versions.yaml — track all component versions
talos: v1.9.2
kubernetes: v1.32.1
components:
  cilium: 1.16.5
  rook-ceph: 1.15.2
  rook-ceph-cluster: 1.15.2
  ingress-nginx: 4.11.3
  cert-manager: 1.16.2
  prometheus-stack: 65.1.0
  external-dns: 1.15.0
```

Update this file alongside every upgrade and store it in version control.

---

## CRD Management

Helm intentionally does not upgrade CRDs on `helm upgrade`. This is a safety measure, but it means you must manage CRD upgrades manually.

```bash
# Option 1: Apply CRDs from the chart before upgrading
# Most charts ship CRDs in a crds/ directory
helm pull cilium/cilium --version 1.16.5 --untar
kubectl apply -f cilium/crds/

# Option 2: Apply CRDs from upstream URLs (component-specific)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.crds.yaml

# Option 3: Use helm with --skip-crds=false (only works on install, not upgrade)
# For upgrades, you must use Option 1 or 2
```

**Warning:** Removing a CRD deletes all custom resources of that type. Never `kubectl delete crd` unless you intend to destroy all instances.
