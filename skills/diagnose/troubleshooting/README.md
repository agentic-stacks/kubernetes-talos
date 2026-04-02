# Diagnose — Troubleshooting Guide

Agent reference for diagnosing Talos Linux and Kubernetes cluster issues. Use this before attempting any remediation.

---

## Safety Rules

1. **Never remediate without explicit user approval** — diagnose first, propose a fix, wait for confirmation.
2. **Verify state before and after** every change — run the relevant diagnostic command before and after applying any fix.
3. **Use the least destructive option** — prefer `talosctl apply-config` over `talosctl reset`; prefer rolling restarts over full cluster restarts.
4. **Never run `talosctl reset` on a control plane node** without understanding the etcd quorum implications.
5. **Always capture a support bundle** before destructive operations — `talosctl support` creates a zip of all node state.
6. **Document what you changed** — if you apply a config patch, record the patch content and the node it was applied to.

---

## Diagnostic Toolkit

These are the essential commands for investigating cluster health. Run them against specific nodes using `-n <node-ip>` or against all endpoints using `-n <endpoint1>,<endpoint2>`.

| # | Command | Purpose | When to Use |
|---|---------|---------|-------------|
| 1 | `talosctl health --nodes <cp1>,<cp2>,<cp3>` | End-to-end cluster health check: etcd, kubelet, apiserver, all nodes ready | First command for any cluster issue |
| 2 | `talosctl services -n <node>` | List all Talos services and their states (Running, Pre, Waiting) | Node not joining, services stuck |
| 3 | `talosctl logs <service> -n <node>` | Stream logs from a Talos service (etcd, kubelet, apid, machined, trustd) | Any service in non-Running state |
| 4 | `talosctl dmesg -n <node>` | Kernel ring buffer — hardware errors, OOM kills, network driver issues | Hardware issues, boot failures |
| 5 | `talosctl get members -n <node>` | Talos discovery membership — which nodes the cluster knows about | Nodes missing from cluster |
| 6 | `talosctl etcd members -n <cp-node>` | etcd cluster membership — lists all etcd peers by name and ID | etcd quorum issues, split brain |
| 7 | `talosctl etcd alarm list -n <cp-node>` | Check for etcd alarms (NOSPACE, CORRUPT) | etcd errors, cluster read-only |
| 8 | `talosctl etcd status -n <cp-node>` | etcd cluster status — DB size, leader, raft term, raft index | Performance issues, DB growth |
| 9 | `talosctl containers --kubernetes -n <node>` | List Kubernetes containers running on the node via CRI | Pod scheduling, container crash loops |
| 10 | `talosctl get staticpods -n <cp-node>` | List static pod manifests managed by Talos (kube-apiserver, etcd, etc.) | Control plane pod issues |
| 11 | `talosctl get manifests -n <cp-node>` | List bootstrap manifests Talos applies (kube-proxy, CoreDNS, CNI) | Missing bootstrap components |
| 12 | `talosctl support -n <node> -O /tmp/support.zip` | Collect full support bundle — logs, configs, state, kernel info | Before destructive changes, filing bug reports |
| 13 | `talosctl netstat -n <node>` | Show network connections on the node — listening ports, established connections | Network connectivity, port conflicts |
| 14 | `talosctl pcap -n <node> --interface eth0 --duration 10s -O /tmp/capture.pcap` | Capture network packets on a node interface | Deep network debugging, traffic analysis |
| 15 | `talosctl read /proc/meminfo -n <node>` | Read system files from the node | Memory pressure, resource investigation |
| 16 | `talosctl get machineconfig -n <node> -o yaml` | Dump the running machine configuration | Config drift, verifying applied patches |

---

## Symptom Decision Trees

### 1. Nodes NotReady

```
kubectl get nodes shows NotReady
|
+-> Is the CNI running?
|   kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium
|   (or check for flannel/calico pods)
|   |
|   +-> No CNI pods -> Was cni.name set to "none"? Install CNI first.
|   +-> CNI pods CrashLooping -> Check CNI pod logs:
|       kubectl logs -n kube-system <cni-pod> --previous
|
+-> Is kubelet running?
|   talosctl services -n <node> | grep kubelet
|   |
|   +-> kubelet not Running -> Check kubelet logs:
|       talosctl logs kubelet -n <node>
|       Common: certificate errors, API server unreachable
|
+-> Is the node network reachable?
|   talosctl get members -n <cp-node>
|   |
|   +-> Node missing from members -> Check node network config:
|       talosctl get addresses -n <node>
|       talosctl get routes -n <node>
|
+-> Is the machine config correct?
    talosctl get machineconfig -n <node> -o yaml
    Check: cluster.controlPlane.endpoint, cluster.network settings
```

### 2. Cannot Reach Talos API

```
talosctl health fails or talosctl version -n <node> times out
|
+-> Is the node reachable on port 50000?
|   (from your workstation) nc -zv <node-ip> 50000
|   |
|   +-> Port closed -> Node may not have booted, check console/IPMI
|   +-> Port open but TLS error -> Continue below
|
+-> Are your talosconfig endpoints correct?
|   cat ~/.talos/config | grep -A5 endpoints
|   Endpoints must list control plane IPs (not worker IPs)
|   |
|   +-> Using VIP as endpoint? VIP only works for K8s API, NOT Talos API.
|       Talos API endpoints must be individual node IPs.
|
+-> Are certSANs configured correctly?
|   talosctl get machineconfig -n <node> -o yaml | grep -A5 certSANs
|   certSANs must include all IPs/hostnames used to reach the node
|   |
|   +-> Missing IP/hostname -> Patch config:
|       talosctl apply-config --nodes <node> --file updated-config.yaml
|
+-> Is apid running?
    talosctl services -n <node> (may need console access)
    Check: apid should be "Running"
```

### 3. etcd Stuck in Pre State

```
talosctl services -n <cp-node> shows etcd in "Pre" state
|
+-> Is this the first control plane node?
|   |
|   +-> Yes -> Has bootstrap been run?
|       talosctl bootstrap -n <first-cp-node>
|       Bootstrap must be run exactly ONCE on exactly ONE control plane node.
|       Running it on multiple nodes causes split-brain.
|
+-> Is this a subsequent control plane node?
|   |
|   +-> Can it reach port 2380 on existing CP nodes?
|       talosctl netstat -n <existing-cp> | grep 2380
|       Firewall must allow TCP 2380 between all CP nodes.
|   |
|   +-> Is there stale etcd data?
|       If node was previously part of a different cluster:
|       talosctl reset --graceful=false -n <node>
|       WARNING: This wipes the node. Requires user approval.
|
+-> Check etcd logs for specific error:
    talosctl logs etcd -n <cp-node>
    Common errors:
    - "member already exists" -> stale member in etcd, remove it first
    - "cluster ID mismatch" -> node has data from different cluster
```

### 4. Bootstrap Stuck at 18/19

```
talosctl bootstrap appears to hang at phase 18 or 19
|
+-> This is EXPECTED when cni.name is set to "none"
|   Phase 18-19 waits for all nodes to become Ready.
|   Nodes will not become Ready until a CNI is installed.
|
+-> Install your CNI immediately after bootstrap:
|   For Cilium:
|     cilium install --set cgroup.autoMount.enabled=false \
|       --set cgroup.hostRoot=/sys/fs/cgroup \
|       --set ipam.mode=kubernetes \
|       --set kubeProxyReplacement=true \
|       --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
|       --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}"
|
+-> If you do NOT install CNI within ~10 minutes:
|   Talos will reboot the node automatically.
|   This is normal. Install the CNI and the node will stabilize.
|
+-> After CNI install, monitor convergence:
    cilium status --wait   (for Cilium)
    kubectl get pods -n kube-system -w
    talosctl health --nodes <cp1>,<cp2>,<cp3>
```

### 5. etcd NOSPACE Alarm

```
etcd returns "etcdserver: mvcc: database space exceeded"
or talosctl etcd alarm list shows NOSPACE
|
+-> Check current DB size:
|   talosctl etcd status -n <cp-node>
|   Default quota is 2GB (2147483648 bytes).
|
+-> If DB is legitimately large (many resources, frequent updates):
|   Increase quota in machine config:
|   cluster:
|     etcd:
|       extraArgs:
|         quota-backend-bytes: "4294967296"   # 4GB
|         auto-compaction-mode: periodic
|         auto-compaction-retention: "5m"
|
+-> Compact and defragment:
|   These commands require kubectl exec into the etcd pod or
|   using talosctl to interact with etcd:
|   talosctl etcd defrag -n <cp-node>
|
+-> Disarm the alarm after freeing space:
|   talosctl etcd alarm disarm -n <cp-node>
|
+-> Verify recovery:
    talosctl etcd alarm list -n <cp-node>   (should be empty)
    talosctl etcd status -n <cp-node>        (DB size should be smaller)
```

### 6. Node Not Joining Cluster

```
A new node boots but does not appear in kubectl get nodes
|
+-> Is Talos discovery working?
|   talosctl get members -n <existing-cp>
|   New node should appear in members list.
|   |
|   +-> Not in members -> Check discovery settings:
|       talosctl get machineconfig -n <new-node> -o yaml | grep -A5 discovery
|       Default: discovery via discovery.talos.dev service
|       Ensure cluster.discovery.enabled is true (default)
|
+-> Do the cluster secrets match?
|   The new node's machine config must have been generated from the
|   SAME secrets as the existing cluster. If you generated a fresh
|   config with talosctl gen config, the secrets will differ.
|   |
|   +-> Secrets mismatch -> Regenerate config using existing secrets:
|       talosctl gen config <cluster-name> <endpoint> \
|         --with-secrets <original-secrets.yaml>
|
+-> Is the node on the same network?
|   talosctl get addresses -n <new-node>
|   talosctl get routes -n <new-node>
|   Node must be able to reach the control plane endpoint.
|   |
|   +-> Test connectivity:
|       talosctl netstat -n <new-node>
|       Look for established connections to port 50001 (trustd) and 6443 (API)
```

### 7. Certificate Errors

```
TLS/certificate errors in talosctl or kubectl output
|
+-> Are certSANs configured for all access methods?
|   talosctl get machineconfig -n <cp-node> -o yaml | grep -A10 certSANs
|   |
|   certSANs must include:
|   - All control plane node IPs
|   - The VIP (if using Talos VIP)
|   - Any load balancer IP/hostname
|   - Any DNS names used to reach the cluster
|   |
|   +-> Missing entries -> Patch config:
|       talosctl patch mc -n <cp-node> --patch '[
|         {"op": "add", "path": "/machine/certSANs/-", "value": "<missing-ip>"}
|       ]'
|
+-> Is there a config mismatch between nodes?
|   Compare certSANs across all control plane nodes:
|   for node in <cp1> <cp2> <cp3>; do
|     echo "=== $node ==="
|     talosctl get machineconfig -n $node -o yaml | grep -A10 certSANs
|   done
|
+-> Has the CA been rotated or is it expired?
|   talosctl get machineconfig -n <cp-node> -o yaml | grep -A2 "ca:"
|   Check certificate validity:
|   talosctl read /system/secrets/ca.crt -n <cp-node> | openssl x509 -noout -dates
|   |
|   +-> CA rotation needed:
|       This is a complex operation. See Talos docs on CA rotation.
|       It requires generating new secrets and rolling all nodes.
```

### 8. Stale Members After Rebuild

```
After rebuilding/reimaging nodes, old members appear in etcd or discovery
|
+-> CRITICAL: Never reuse cluster secrets from a destroyed cluster
|   with the same node names/IPs unless you have cleaned up etcd.
|   Old members with the same identity will conflict.
|
+-> Stale discovery members (TTL-based):
|   Talos discovery entries have a 30-minute TTL.
|   If you destroyed a cluster and are rebuilding:
|   - Wait 30 minutes for stale entries to expire, OR
|   - Use a different cluster name/discovery ID
|   |
|   Check current members:
|   talosctl get members -n <cp-node>
|
+-> Stale etcd members:
|   If an etcd member was removed but its entry persists:
|   talosctl etcd members -n <cp-node>
|   talosctl etcd remove-member <member-id> -n <cp-node>
|   WARNING: Only remove members that are genuinely gone.
|   Removing a live member causes data loss.
|
+-> Full cluster reset (nuclear option):
    If the cluster is unsalvageable:
    talosctl reset --graceful=false -n <node>
    This must be run on EVERY node. Then re-bootstrap from scratch.
    REQUIRES EXPLICIT USER APPROVAL.
```

---

## Gathering a Support Bundle

The `talosctl support` command collects comprehensive diagnostic data from one or more nodes into a zip archive. Always gather this before filing bug reports or before performing destructive operations.

```bash
# Single node
talosctl support -n <node> -O /tmp/support-node1.zip

# Multiple nodes
talosctl support -n <cp1>,<cp2>,<cp3>,<w1>,<w2> -O /tmp/support-full.zip

# Specific number of log lines (default is 500)
talosctl support -n <node> --num-tail 1000 -O /tmp/support-verbose.zip
```

The support bundle includes:
- Talos service logs (machined, apid, etcd, kubelet, trustd, containerd)
- Machine configuration (secrets are redacted)
- Kernel ring buffer (dmesg)
- Resource state (members, addresses, routes, links)
- etcd status and member list
- Kubernetes pod list and events
- Controller and resource versions

---

## Quick Reference: Service ↔ Port Mapping

| Service | Port | Protocol | Notes |
|---------|------|----------|-------|
| apid (Talos API) | 50000 | gRPC/TLS | Must be reachable from workstation |
| trustd | 50001 | gRPC/TLS | Inter-node trust during bootstrap |
| etcd client | 2379 | gRPC/TLS | Kubernetes API server to etcd |
| etcd peer | 2380 | gRPC/TLS | etcd node-to-node replication |
| Kubernetes API | 6443 | HTTPS | kubectl, VIP, load balancer target |
| kubelet | 10250 | HTTPS | API server to kubelet communication |
| kubelet healthz | 10248 | HTTP | Local health checks only |
