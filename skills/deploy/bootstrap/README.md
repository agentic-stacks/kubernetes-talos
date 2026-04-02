# Deploy: Bootstrap

Bootstrap is the process of taking bare-metal (or VM) nodes booted from Talos media and turning them into a running Kubernetes cluster. This skill covers everything from first boot to a healthy `kubectl get nodes` output.

---

## Prerequisites Checklist

Before issuing a single `talosctl` command, verify every item below.

- **`talosctl` installed and version-matched.** The version of `talosctl` used to generate configs controls which version of Talos Linux is installed — NOT the version of the ISO/PXE image the nodes booted from. Run `talosctl version` and confirm the client version matches your intended cluster version.
- **Nodes booted from Talos ISO or PXE.** Talos runs entirely in RAM until a machine config is applied. The console will show a maintenance screen with the node's IP address and a message that it is waiting for configuration.
- **Network ports open (TCP):**
  - `50000` — Talos API (required from your workstation to all nodes during bootstrap, then only to control-plane nodes afterwards)
  - `50001` — Talos trustd (inter-node certificate bootstrapping)
  - `6443` — Kubernetes API server (from workstation and between nodes)
  - `2379–2380` — etcd client/peer traffic (between control-plane nodes only)
- **Control-plane endpoint decided and reachable.** This is the stable HTTPS address Kubernetes components and `kubectl` will use long-term. Options:
  - Virtual IP (VIP) managed by Talos — simplest for bare-metal
  - External load balancer (HAProxy, cloud LB, etc.)
  - DNS name resolving to one or more CP node IPs
  - For single-node dev clusters: the node IP directly

  The endpoint must be reachable as `https://<endpoint>:6443` before you run `talosctl gen config`.

---

## Bootstrap Procedure

> Bootstrap must be run exactly **once**, on exactly **one** control-plane node. Running it more than once corrupts the etcd cluster.

```
Step 1: Identify node IPs from console output or DHCP logs

  Each node prints its IP at the maintenance screen.
  Alternatively check your DHCP server's lease table.
  Example IPs used below:
    Control plane 1: 192.168.1.10
    Control plane 2: 192.168.1.11
    Control plane 3: 192.168.1.12
    Worker 1:        192.168.1.20
    Worker 2:        192.168.1.21
    VIP (endpoint):  192.168.1.100

Step 2: Generate secrets bundle

  talosctl gen secrets -o secrets.yaml

  Keep secrets.yaml safe — it contains the CA keys for your cluster.
  Do not commit it to version control.

Step 3: Generate machine configs

  talosctl gen config my-cluster https://192.168.1.100:6443 \
    --with-secrets secrets.yaml

  Produces: controlplane.yaml, worker.yaml, talosconfig
  The endpoint URL becomes baked into all generated certificates.

Step 4: (Optional) Customise configs before applying

  Edit controlplane.yaml and worker.yaml as needed:
    - Set the correct install disk (default /dev/sda; use /dev/vda for VMs)
    - Add certSANs if nodes need extra names in the API server cert
    - Enable VIP if using Talos-managed virtual IP
    - Set allowSchedulingOnControlPlanes: true for single-node clusters

  Verify the install disk available on a node:
    talosctl -n 192.168.1.10 get disks --insecure

Step 5: Apply config to each control-plane node

  talosctl apply-config --insecure --nodes 192.168.1.10 --file controlplane.yaml
  talosctl apply-config --insecure --nodes 192.168.1.11 --file controlplane.yaml
  talosctl apply-config --insecure --nodes 192.168.1.12 --file controlplane.yaml

  --insecure is required pre-PKI: the connection is encrypted but unauthenticated
  because the node has no client certificate to verify yet. Once a config is
  applied the node generates its PKI and subsequent connections use mutual TLS.

Step 6: Apply config to each worker node

  talosctl apply-config --insecure --nodes 192.168.1.20 --file worker.yaml
  talosctl apply-config --insecure --nodes 192.168.1.21 --file worker.yaml

Step 7: Configure talosctl client context

  talosctl config endpoint 192.168.1.10 192.168.1.11 192.168.1.12
  talosctl config node 192.168.1.10

  This writes into ~/.talos/config (or the file pointed to by TALOSCONFIG).
  Endpoints are the CP nodes talosctl connects to. Node is the default target.

Step 8: Bootstrap etcd (ONCE, on ONE control-plane node only)

  talosctl bootstrap --nodes 192.168.1.10

  This tells the first CP node to initialise etcd as a single-member cluster.
  The other CP nodes will join etcd automatically once they can reach node 1.

Step 9: Wait for the control plane to come up

  Watch progress with:
    talosctl health --nodes 192.168.1.10,192.168.1.11,192.168.1.12

  Expected sequence:
    etcd → kube-apiserver → kube-controller-manager → kube-scheduler

  This typically takes 2–5 minutes. etcd must be healthy before the API server
  starts. The API server must be healthy before nodes register.

Step 10: Retrieve kubeconfig

  talosctl kubeconfig

  Merges the cluster's kubeconfig into ~/.kube/config.
  Use --force to overwrite an existing entry with the same cluster name.

Step 11: Verify the cluster

  kubectl get nodes
  kubectl get pods -n kube-system
  talosctl health --nodes 192.168.1.10,192.168.1.11,192.168.1.12
```

---

## Endpoints vs. Nodes

This is the most common source of confusion with `talosctl`.

**Endpoints** are the control-plane nodes that `talosctl` establishes a direct connection to. They act as a proxy: `talosctl` connects to an endpoint, and the endpoint forwards the request to the target node over the cluster's internal network. You can specify multiple endpoints for failover.

**Nodes** are the targets of an operation — the machines you want to inspect or modify. A node can be any cluster member, including workers, and does not need to be reachable directly from your workstation once the cluster is up.

```
Your workstation
    │
    │  TCP 50000
    ▼
Endpoint (CP node: 192.168.1.10)
    │
    │  proxied internally
    ▼
Target node (worker: 192.168.1.20)
```

Practical rules:
- **NEVER use a VIP as a Talos API endpoint.** The VIP is for the Kubernetes API (port 6443). If the current VIP holder goes down, `talosctl` connectivity to the cluster would be lost. Use individual CP node IPs as Talos endpoints.
- **NEVER use worker nodes as endpoints.** Workers cannot proxy requests to other nodes.
- Set multiple endpoints so `talosctl` can fail over: `talosctl config endpoint 192.168.1.10 192.168.1.11 192.168.1.12`

Inline flags (override config for a single command):
```bash
# Route through endpoint .10, target node .20
talosctl -e 192.168.1.10 -n 192.168.1.20 dmesg

# Target multiple nodes in one command
talosctl -e 192.168.1.10 -n 192.168.1.20,192.168.1.21 services
```

The `-e` / `--endpoints` and `-n` / `--nodes` flags are independent; mixing them lets you reach any node through any reachable CP.

---

## Single-Node Dev Clusters

For local development and testing, a single node can run both control-plane and workload pods.

**Option A: Bare-metal or VM single node**

Add to `controlplane.yaml` under `machine.kubelet` / cluster patch (or use `--config-patch` at gen time):

```yaml
cluster:
  allowSchedulingOnControlPlanes: true
```

Then follow the standard bootstrap procedure with a single CP IP. There is no HA: if the node goes down, etcd is lost and the cluster must be bootstrapped again. Acceptable for dev, not for production.

**Option B: Docker-based local cluster**

`talosctl cluster create` spins up a fully functional Talos cluster using Docker containers — no VMs required:

```bash
# Single-node cluster
talosctl cluster create

# Multi-node cluster
talosctl cluster create --controlplanes 1 --workers 2

# Tear down
talosctl cluster destroy
```

This is the fastest way to iterate on Talos configs and test workloads locally.

---

## Multi-Control-Plane HA

For production clusters use an odd number of control-plane nodes (3 is standard, 5 for very large clusters) to maintain etcd quorum. With 3 nodes you can tolerate 1 failure; with 5 you can tolerate 2.

**Etcd quorum formula:** `(n/2) + 1` nodes must be healthy. With 2 CPs you have NO fault tolerance — losing one node loses quorum.

### VIP (Talos-managed Virtual IP)

Talos can manage a shared VIP on the control-plane network interface. The current VIP holder is elected using ARP. Configure in `controlplane.yaml`:

```yaml
machine:
  network:
    interfaces:
      - interface: eth0
        dhcp: true
        vip:
          ip: 192.168.1.100
```

Generate config pointing at the VIP as the Kubernetes endpoint:
```bash
talosctl gen config my-cluster https://192.168.1.100:6443 --with-secrets secrets.yaml
```

Remember: use individual CP node IPs (not the VIP) as `talosctl` endpoints.

### External Load Balancer

For cloud deployments or when a VIP is not suitable:
- HAProxy, nginx, AWS NLB, GCP TCP LB, etc.
- Forward TCP 6443 to all CP nodes
- Health-check on TCP 6443
- The LB DNS name or IP becomes the `talosctl gen config` endpoint

### Verify etcd quorum after bootstrap

```bash
talosctl -n 192.168.1.10 etcd members
```

Expected output shows all CP node IPs as `started` members. If a node shows as `unstarted`, it has not yet joined etcd — check inter-node connectivity on port 2380.

Adding additional CP nodes after initial bootstrap: apply `controlplane.yaml` with `--insecure`; they join etcd automatically.

---

## Post-Bootstrap Validation

Run these checks before declaring the cluster ready:

```bash
# Full cluster health check (waits until all components are healthy)
talosctl health --nodes 192.168.1.10,192.168.1.11,192.168.1.12

# Node registration
kubectl get nodes -o wide

# System pod health (coredns, kube-proxy, CNI daemonsets)
kubectl get pods -n kube-system

# etcd member list (all CPs should appear as started)
talosctl -n 192.168.1.10 etcd members

# Check individual node service status
talosctl -n 192.168.1.10 services
talosctl -n 192.168.1.11 services
talosctl -n 192.168.1.12 services

# View recent node logs for any CP node
talosctl -n 192.168.1.10 dmesg | tail -50
```

A healthy cluster shows:
- All nodes in `Ready` state (may be `NotReady` until CNI is installed)
- `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` pods `Running`
- etcd pods `Running` on all CP nodes
- `talosctl health` exits 0

---

## Common Bootstrap Failures

### "Stuck at phase 18/19 — nodes remain NotReady"

**Symptom:** `kubectl get nodes` shows all nodes as `NotReady` indefinitely after bootstrap completes.

**Decision tree:**
1. Is this expected? If you set `cni: none` in the cluster config or are using a custom CNI, nodes will remain `NotReady` until you install the CNI. This is correct behavior.
2. Install your CNI (Cilium, Flannel, Calico, etc.) and nodes will transition to `Ready` within ~60 seconds.
3. If CNI is installed and nodes are still `NotReady`, check CNI pods: `kubectl get pods -n kube-system` — look for CrashLoopBackOff or Pending pods.
4. Check kubelet logs: `talosctl -n 192.168.1.10 logs kubelet`

### "etcd services in Pre state / etcd not starting"

**Symptom:** `talosctl services` shows etcd in `Pre` or `Preparing` state and it never transitions to `Running`.

**Decision tree:**
1. Did you run `talosctl bootstrap`? Without it, CP nodes wait indefinitely for the bootstrap signal. Run it now: `talosctl bootstrap --nodes 192.168.1.10`
2. Is port 2380 open between CP nodes? etcd peers need TCP 2380. Test with: `talosctl -n 192.168.1.11 get addresses` and verify routing.
3. Is there old etcd data on disk from a previous attempt? Reset the ephemeral partition: `talosctl reset --nodes 192.168.1.11 --system-labels-to-wipe EPHEMERAL` then reapply the config and retry. Do NOT reset the bootstrapping node unless all nodes are being reset.
4. Check etcd logs: `talosctl -n 192.168.1.10 logs etcd`

### "Certificate errors / x509 errors connecting"

**Symptom:** `talosctl` or `kubectl` commands fail with TLS/certificate errors.

**Decision tree:**
1. **certSANs missing an access IP or hostname.** If you access the API server via an IP not in the cert's SANs, add it to `controlplane.yaml` under `cluster.apiServer.certSANs` and reapply.
2. **talosconfig mismatch.** The `talosconfig` file must have been generated from the same secrets as the machine configs. If you regenerated secrets and redeployed, regenerate talosconfig too: `talosctl gen config ...` and use the new talosconfig.
3. **Wrong endpoint URL format.** The endpoint must be `https://` with port 6443. `http://` or omitting the port causes cert validation failures.
4. **Client cert expired or from wrong CA.** Delete and re-fetch: `talosctl kubeconfig --force`

### "Node not found during bootstrap" / API server errors immediately after bootstrap

**Symptom:** Shortly after `talosctl bootstrap`, you see errors about nodes not found or API server refusing connections.

**Resolution:** This is normal and transient. The API server is still starting up when the bootstrap signal is processed. Wait 60–120 seconds and retry. `talosctl health` will tell you when the API server is stable.

### "Configuration validation failed: specified install disk does not exist"

**Symptom:** `talosctl apply-config` returns a validation error about `/dev/sda`.

**Resolution:** The node does not have a disk at the default path. Find the correct disk:
```bash
talosctl -n 192.168.1.10 get disks --insecure
```
Then patch `controlplane.yaml` or `worker.yaml` to set `machine.install.disk` to the correct path (e.g., `/dev/vda` for QEMU/KVM VMs, `/dev/nvme0n1` for NVMe drives).
