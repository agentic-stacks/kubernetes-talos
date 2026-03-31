# Operations: Health Check

Procedures for verifying cluster health across every layer — Talos OS, etcd, Kubernetes control plane, and workloads. Use before and after any operational change.

---

## Quick Health Check (60 seconds)

Run these five commands in sequence. If all pass, the cluster is healthy.

```bash
# 1. Talos-level health (checks etcd, kubelet, apid, all nodes)
#    Defaults to 3-minute timeout; override with --wait-timeout
talosctl health --nodes 192.168.1.10

# 2. etcd cluster status — every member should report "started"
talosctl etcd status --nodes 192.168.1.10

# 3. etcd alarm list — must return empty (no alarms)
talosctl etcd alarm list --nodes 192.168.1.10

# 4. All Kubernetes nodes should be Ready
kubectl get nodes -o wide

# 5. All kube-system pods should be Running or Completed
kubectl get pods -n kube-system -o wide
```

If any command fails, proceed to the comprehensive check to isolate the layer.

---

## Comprehensive Health Check

Work through each layer top-to-bottom. Stop at the first layer that shows problems — fixing lower layers often resolves higher-layer symptoms.

### Layer 1: Talos OS

```bash
# Service status on every node — all services should be "Running"
talosctl services --nodes 192.168.1.10,192.168.1.11,192.168.1.12

# System dmesg — look for OOM kills, disk errors, NIC resets
talosctl dmesg --nodes 192.168.1.10 | tail -100

# Disk usage — Talos partitions must not be full
talosctl usage /var --nodes 192.168.1.10

# Memory pressure
talosctl memory --nodes 192.168.1.10

# CPU load
talosctl stats --nodes 192.168.1.10

# System time sync — clock skew breaks etcd and certificates
talosctl time --nodes 192.168.1.10,192.168.1.11,192.168.1.12

# Talos version on each node — should all match
talosctl version --nodes 192.168.1.10,192.168.1.11,192.168.1.12
```

### Layer 2: etcd

```bash
# Member list — all members should be "started", no learners stuck
talosctl etcd members --nodes 192.168.1.10

# Cluster status with leader, term, and index info
talosctl etcd status --nodes 192.168.1.10

# Alarms — NOSPACE alarm means etcd DB hit quota (default 2 GiB)
talosctl etcd alarm list --nodes 192.168.1.10

# etcd DB size — should be well under 2 GiB for normal clusters
# If size is large, check for excessive events or secrets
talosctl etcd status --nodes 192.168.1.10 | grep -i "db size"

# Defrag if DB size is high (run on one node at a time)
talosctl etcd defrag --nodes 192.168.1.10
```

### Layer 3: Kubernetes Control Plane

```bash
# Node status
kubectl get nodes -o wide

# Control-plane component health
kubectl get componentstatuses 2>/dev/null || echo "componentstatuses deprecated in newer K8s — use pod checks"
kubectl get pods -n kube-system -o wide

# API server responsiveness — should return fast
kubectl get --raw /healthz

# Scheduler and controller manager
kubectl get pods -n kube-system -l component=kube-scheduler -o wide
kubectl get pods -n kube-system -l component=kube-controller-manager -o wide

# Check for any pods in bad states cluster-wide
kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded
```

### Layer 4: Platform Components

```bash
# CNI (Cilium example)
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium -o wide
cilium status 2>/dev/null || kubectl exec -n kube-system ds/cilium -- cilium status --brief

# CSI / Storage
kubectl get pods -n kube-system -l app=rook-ceph-operator -o wide 2>/dev/null
kubectl get sc
kubectl get pv --sort-by=.status.phase

# CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default.svc.cluster.local

# Ingress controller
kubectl get pods -n ingress-nginx -o wide 2>/dev/null
```

### Layer 5: Workload Health

```bash
# Deployments not at desired replica count
kubectl get deployments --all-namespaces | awk '$3 != $4'

# StatefulSets not at desired replica count
kubectl get statefulsets --all-namespaces | awk '$3 != $4'

# DaemonSets with unavailable pods
kubectl get daemonsets --all-namespaces | awk '$4 != $5'

# Persistent volume claims stuck in Pending
kubectl get pvc --all-namespaces --field-selector=status.phase=Pending

# Recent events cluster-wide (last 15 minutes)
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -30

# Resource pressure on nodes
kubectl top nodes 2>/dev/null
kubectl describe nodes | grep -A 5 "Conditions:"
```

---

## Health Report Format

When reporting cluster health, use this structured format:

```
CLUSTER HEALTH REPORT
=====================
Cluster:     my-cluster
Timestamp:   2025-01-15T10:30:00Z
Talos:       v1.9.2
Kubernetes:  v1.32.1

Overall Status: HEALTHY | DEGRADED | UNHEALTHY

Layer 1 - Talos OS:           OK | WARN | FAIL
  - Services:                 OK (all running)
  - Disk:                     OK (42% used)
  - Memory:                   OK (6.2/16 GiB)
  - Time sync:                OK (offset <1ms)

Layer 2 - etcd:               OK | WARN | FAIL
  - Members:                  3/3 started
  - Leader:                   node-1 (term 47)
  - DB size:                  28 MiB / 2 GiB
  - Alarms:                   none

Layer 3 - Kubernetes:         OK | WARN | FAIL
  - Nodes:                    3 CP + 2 W (all Ready)
  - API server:               responding (<50ms)
  - System pods:              14/14 Running

Layer 4 - Platform:           OK | WARN | FAIL
  - CNI (Cilium):             OK
  - DNS (CoreDNS):            OK
  - Storage:                  OK (3 PVs bound)

Layer 5 - Workloads:          OK | WARN | FAIL
  - Deployments:              8/8 at desired count
  - StatefulSets:             2/2 at desired count
  - Failed pods:              0
  - Pending PVCs:             0

Issues:
  (none)  — or list each issue with layer and recommended action
```

**Status definitions:**
- **HEALTHY** — all layers OK, no issues detected
- **DEGRADED** — cluster is functional but one or more components are in a warning state (e.g., a single worker NotReady, high disk usage)
- **UNHEALTHY** — cluster functionality is impaired (e.g., etcd quorum lost, API server unreachable, multiple nodes down)

---

## Automated Health Script

Save to `scripts/health-check.sh` for quick one-command checks:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration — adjust to match your cluster
CP_NODES="192.168.1.10,192.168.1.11,192.168.1.12"
FIRST_CP="192.168.1.10"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

pass() { echo -e "  ${GREEN}PASS${NC}  $1"; }
warn() { echo -e "  ${YELLOW}WARN${NC}  $1"; }
fail() { echo -e "  ${RED}FAIL${NC}  $1"; ERRORS=$((ERRORS + 1)); }

ERRORS=0

echo "=== Cluster Health Check ==="
echo "Timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
echo ""

# Talos health (overall)
echo "--- Talos Health ---"
if talosctl health --nodes "$FIRST_CP" --wait-timeout 60s 2>/dev/null; then
    pass "talosctl health"
else
    fail "talosctl health reported errors"
fi

# etcd
echo "--- etcd ---"
if talosctl etcd status --nodes "$FIRST_CP" >/dev/null 2>&1; then
    pass "etcd cluster status"
else
    fail "etcd cluster unreachable"
fi

ALARMS=$(talosctl etcd alarm list --nodes "$FIRST_CP" 2>/dev/null)
if [ -z "$ALARMS" ] || echo "$ALARMS" | grep -q "no alarms"; then
    pass "etcd alarms (none)"
else
    fail "etcd alarms active: $ALARMS"
fi

# Kubernetes nodes
echo "--- Kubernetes Nodes ---"
NOT_READY=$(kubectl get nodes --no-headers 2>/dev/null | grep -v " Ready " || true)
if [ -z "$NOT_READY" ]; then
    NODE_COUNT=$(kubectl get nodes --no-headers 2>/dev/null | wc -l | tr -d ' ')
    pass "all $NODE_COUNT nodes Ready"
else
    fail "nodes not Ready: $(echo "$NOT_READY" | awk '{print $1}' | tr '\n' ' ')"
fi

# kube-system pods
echo "--- kube-system Pods ---"
BAD_PODS=$(kubectl get pods -n kube-system --no-headers 2>/dev/null | grep -v -E "Running|Completed" || true)
if [ -z "$BAD_PODS" ]; then
    pass "all kube-system pods healthy"
else
    fail "unhealthy kube-system pods: $(echo "$BAD_PODS" | awk '{print $1}' | tr '\n' ' ')"
fi

# Failed pods cluster-wide
echo "--- Workloads ---"
FAILED=$(kubectl get pods --all-namespaces --no-headers --field-selector=status.phase=Failed 2>/dev/null | wc -l | tr -d ' ')
PENDING=$(kubectl get pods --all-namespaces --no-headers --field-selector=status.phase=Pending 2>/dev/null | wc -l | tr -d ' ')
if [ "$FAILED" -eq 0 ] && [ "$PENDING" -eq 0 ]; then
    pass "no Failed or Pending pods"
else
    [ "$FAILED" -gt 0 ] && warn "$FAILED Failed pods"
    [ "$PENDING" -gt 0 ] && warn "$PENDING Pending pods"
fi

echo ""
if [ "$ERRORS" -eq 0 ]; then
    echo -e "Overall: ${GREEN}HEALTHY${NC}"
else
    echo -e "Overall: ${RED}UNHEALTHY${NC} ($ERRORS checks failed)"
    exit 1
fi
```

```bash
chmod +x scripts/health-check.sh
./scripts/health-check.sh
```

---

## When to Run Health Checks

| Trigger | Check Level | Reason |
|---------|-------------|--------|
| Before any upgrade | Comprehensive | Ensure baseline health; never upgrade a sick cluster |
| After any upgrade | Comprehensive | Verify upgrade did not introduce regressions |
| Before/after scaling | Quick + etcd | Confirm etcd membership and node registration |
| After config changes | Quick | Verify config applied without side effects |
| Daily (cron) | Quick (scripted) | Detect drift and silent failures early |
| Troubleshooting | Comprehensive | Systematically isolate the failing layer |
| Before taking etcd snapshot | Quick | Ensure you are snapshotting healthy state |
| After disaster recovery | Comprehensive | Full validation that recovery succeeded |
