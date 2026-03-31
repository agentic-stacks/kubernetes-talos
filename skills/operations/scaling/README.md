# Operations: Scaling

Procedures for adding and removing nodes — both workers and control-plane members. Covers topology changes including single-node to HA migration.

---

## Adding Worker Nodes

Workers are stateless from etcd's perspective. Adding them is low-risk.

### Step 1: Generate a Node-Specific Patch

Create a patch file for the new worker with its unique network configuration:

```yaml
# patches/worker-3.yaml
machine:
  network:
    hostname: worker-3
    interfaces:
      - interface: eth0
        dhcp: false
        addresses:
          - 192.168.1.22/24
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.1.1
  nodeLabels:
    topology.kubernetes.io/zone: "rack-2"
```

### Step 2: Generate the Config

```bash
# Generate worker config with the node-specific patch applied
talosctl gen config my-cluster https://192.168.1.100:6443 \
  --with-secrets secrets.yaml \
  --config-patch-worker @patches/worker-3.yaml \
  --output worker-3.yaml \
  --output-types worker
```

Alternatively, if you already have a base `worker.yaml`, apply the patch directly:

```bash
talosctl machineconfig patch worker.yaml \
  --patch @patches/worker-3.yaml \
  --output worker-3.yaml
```

### Step 3: Boot the Node

Boot the new machine from the Talos ISO (or PXE). Wait for it to reach the maintenance screen showing its IP address.

### Step 4: Apply the Config

```bash
# --insecure is required because the node has no config yet (no TLS identity)
talosctl apply-config --insecure \
  --nodes 192.168.1.22 \
  --file worker-3.yaml
```

The node will install to disk, reboot, and join the cluster automatically.

### Step 5: Verify

```bash
# Watch the node appear (may take 1-2 minutes after reboot)
kubectl get nodes -w

# Confirm the node is Ready
kubectl get nodes -o wide | grep worker-3

# Verify Talos services are running
talosctl services --nodes 192.168.1.22

# Check that CNI pods scheduled onto the new node
kubectl get pods -n kube-system -o wide --field-selector spec.nodeName=worker-3
```

---

## Adding Control-Plane Nodes

Control-plane nodes run etcd. Strict rules apply.

**Critical rules:**
- Always maintain an **odd number** of control-plane nodes: 1, 3, 5
- Adding a CP node means etcd membership changes — this affects quorum
- Never add more than one CP node at a time; wait for each to fully join before adding the next

### Step 1: Generate a Node-Specific Patch

```yaml
# patches/cp-3.yaml
machine:
  network:
    hostname: cp-3
    interfaces:
      - interface: eth0
        dhcp: false
        addresses:
          - 192.168.1.12/24
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.1.1
  # If using Talos VIP for the control-plane endpoint:
  network:
    interfaces:
      - interface: eth0
        dhcp: false
        addresses:
          - 192.168.1.12/24
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.1.1
        vip:
          ip: 192.168.1.100
```

### Step 2: Generate and Apply Config

```bash
talosctl gen config my-cluster https://192.168.1.100:6443 \
  --with-secrets secrets.yaml \
  --config-patch-control-plane @patches/cp-3.yaml \
  --output cp-3.yaml \
  --output-types controlplane

# Boot the node from Talos media, then:
talosctl apply-config --insecure \
  --nodes 192.168.1.12 \
  --file cp-3.yaml
```

**Do NOT run `talosctl bootstrap` on the new node.** Bootstrap was already run once during initial cluster creation. The new CP node joins the existing etcd cluster automatically.

### Step 3: Verify etcd Membership

```bash
# Confirm the new node joined etcd — should show all 3 (or 5) members
talosctl etcd members --nodes 192.168.1.10

# Verify etcd health
talosctl etcd status --nodes 192.168.1.10

# Confirm Kubernetes sees the node as a control-plane
kubectl get nodes -l node-role.kubernetes.io/control-plane -o wide
```

### Step 4: Update Load Balancer / DNS

If you use an external load balancer for the API server endpoint (instead of Talos VIP), add the new CP node's IP to the backend pool.

### Step 5: Update talosconfig Endpoints

Your local `talosconfig` should list all CP node IPs so `talosctl` can reach the cluster if any single CP goes down:

```bash
talosctl config endpoints 192.168.1.10 192.168.1.11 192.168.1.12
```

---

## Removing Worker Nodes

### Step 1: Drain the Node

```bash
# Cordon and drain — evicts all pods respecting PDBs
kubectl drain worker-3 --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

### Step 2: Reset the Talos Node

```bash
# Graceful reset — wipes the node's disk and reverts to maintenance mode
talosctl reset --graceful --nodes 192.168.1.22
```

### Step 3: Remove from Kubernetes

```bash
# Delete the node object from the API server
kubectl delete node worker-3

# Verify
kubectl get nodes
```

The physical or virtual machine can now be repurposed or powered off.

---

## Removing Control-Plane Nodes

**Critical safety rules:**
- **Never go below 3 control-plane nodes** in a production HA cluster
- **Never remove more than one CP at a time**
- **Always verify etcd quorum** after removal
- Quorum requires a strict majority: 3 nodes tolerate 1 failure, 5 tolerate 2

### Step 1: Verify Current etcd State

```bash
# Confirm current membership and which node is leader
talosctl etcd members --nodes 192.168.1.10
talosctl etcd status --nodes 192.168.1.10
```

### Step 2: Drain the Node

```bash
kubectl drain cp-3 --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

### Step 3: Remove from etcd

The node should leave etcd gracefully as part of reset. If it does not, force remove it:

```bash
# If graceful leave fails, explicitly remove the member
# First get the member ID from the members list
talosctl etcd members --nodes 192.168.1.10
# Then remove (use the hex member ID from the output)
talosctl etcd remove-member <member-id> --nodes 192.168.1.10
```

### Step 4: Reset the Node

```bash
talosctl reset --graceful --nodes 192.168.1.12
```

### Step 5: Clean Up

```bash
# Delete the node object
kubectl delete node cp-3

# Verify etcd still has quorum
talosctl etcd members --nodes 192.168.1.10
talosctl etcd status --nodes 192.168.1.10

# Update talosconfig endpoints to remove the old IP
talosctl config endpoints 192.168.1.10 192.168.1.11

# Update load balancer / DNS if applicable
```

### Step 6: Post-Removal Health Check

```bash
talosctl health --nodes 192.168.1.10
kubectl get nodes
kubectl get pods -n kube-system
```

---

## Topology Changes: Single-Node to HA Migration

A single control-plane node has no etcd redundancy. Migrating to 3 CP nodes is the most common topology change.

### Pre-Migration State

```
1 CP node (192.168.1.10) — runs etcd, API server, scheduler, controller manager, workloads
```

### Target State

```
3 CP nodes (192.168.1.10, 192.168.1.11, 192.168.1.12) — HA etcd, workloads on workers
+ N worker nodes (optional)
```

### Procedure

**1. Take an etcd snapshot before starting:**

```bash
talosctl etcd snapshot db.snapshot --nodes 192.168.1.10
```

**2. Add the second control-plane node (going from 1 to 2):**

```bash
# Generate and apply config for cp-2 (see "Adding Control-Plane Nodes" above)
talosctl apply-config --insecure --nodes 192.168.1.11 --file cp-2.yaml

# Wait for cp-2 to join etcd
talosctl etcd members --nodes 192.168.1.10
# Should show 2 members
```

**Warning:** A 2-node etcd cluster has **worse** fault tolerance than 1 node (quorum requires both nodes). Move quickly to step 3.

**3. Add the third control-plane node:**

```bash
talosctl apply-config --insecure --nodes 192.168.1.12 --file cp-3.yaml

# Wait for cp-3 to join
talosctl etcd members --nodes 192.168.1.10
# Should show 3 members — you now have HA
```

**4. If the original single node was configured with `cluster.allowSchedulingOnControlPlanes: true`, optionally disable it now and add dedicated workers:**

```yaml
# patch-disable-cp-scheduling.yaml
cluster:
  allowSchedulingOnControlPlanes: false
```

```bash
# Apply to all CP nodes
talosctl patch machineconfig --nodes 192.168.1.10 --patch @patch-disable-cp-scheduling.yaml
talosctl patch machineconfig --nodes 192.168.1.11 --patch @patch-disable-cp-scheduling.yaml
talosctl patch machineconfig --nodes 192.168.1.12 --patch @patch-disable-cp-scheduling.yaml
```

**5. Set up the control-plane endpoint for HA:**

If you were using the single node's IP as the endpoint, switch to a VIP or load balancer:

```yaml
# patch-vip.yaml
machine:
  network:
    interfaces:
      - interface: eth0
        vip:
          ip: 192.168.1.100
```

```bash
# Apply VIP config to all CP nodes
talosctl patch machineconfig --nodes 192.168.1.10 --patch @patch-vip.yaml
talosctl patch machineconfig --nodes 192.168.1.11 --patch @patch-vip.yaml
talosctl patch machineconfig --nodes 192.168.1.12 --patch @patch-vip.yaml

# Update talosconfig and kubeconfig to use the VIP
talosctl config endpoints 192.168.1.10 192.168.1.11 192.168.1.12
talosctl kubeconfig --nodes 192.168.1.10 --force
```

**6. Verify the migration:**

```bash
talosctl health --nodes 192.168.1.10
talosctl etcd members --nodes 192.168.1.10
talosctl etcd status --nodes 192.168.1.10
kubectl get nodes -o wide
```
