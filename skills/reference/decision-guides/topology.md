# Decision Guide — Cluster Topology

## Context

Cluster topology determines the number of nodes, their roles (control plane vs worker), and their resource allocation. This decision is made at **cluster creation** but can be expanded later by adding nodes.

**When to decide:** Before generating machine configs with `talosctl gen config`.

**Can it be changed later?** Partially. You can add worker nodes at any time. Adding control plane nodes requires care (etcd quorum). Removing control plane nodes requires etcd member removal. You cannot convert a worker to a control plane node or vice versa without wiping and re-provisioning.

**Talos-specific note:** Talos generates separate configs for control plane and worker nodes. Each role has distinct machine config types: `controlplane` and `worker`.

---

## Options

### Single-Node Cluster (1 CP)

One node acting as both control plane and worker. Uses `allowSchedulingOnControlPlanes: true`.

**Strengths:**
- Minimum hardware requirement
- Simple to set up and manage
- Suitable for development, CI/CD, and edge deployments

**Weaknesses:**
- No redundancy — node failure = total cluster outage
- etcd has no replication — disk failure = data loss
- Not suitable for any workload requiring availability

**Machine config adjustment:**

```yaml
cluster:
  allowSchedulingOnControlPlanes: true
```

**Resource requirements:**

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Disk (OS) | 10 GB | 20 GB |
| Disk (etcd) | N/A (shared) | Separate 10 GB SSD recommended |

### 3 Control Plane + 3 Workers (3CP + 3W)

Standard HA topology. Three control plane nodes for etcd quorum, three dedicated worker nodes for workloads.

**Strengths:**
- Full HA — tolerates 1 CP and 1 worker failure simultaneously
- etcd has 3 replicas (tolerates 1 failure)
- Clean separation of control plane and workload resources
- Standard topology recommended by Kubernetes documentation

**Weaknesses:**
- Requires 6 nodes minimum
- Control plane nodes are often underutilized if not scheduling workloads

**When to use:** Production clusters with moderate workloads. The most common and well-documented topology.

### 3 Control Plane + N Workers (3CP + NW)

Same as above but with a variable worker pool. Scale workers based on workload demand.

**Strengths:**
- HA control plane with flexible worker scaling
- Workers can be heterogeneous (different sizes, GPUs, etc.)
- Worker nodes can be added/removed without affecting control plane

**Weaknesses:**
- Minimum 3 nodes for control plane regardless of workload size
- Requires monitoring to right-size the worker pool

**When to use:** Production clusters expecting growth. Start with 3CP + 3W and add workers as needed.

### 5 Control Plane + N Workers (5CP + NW)

Five control plane nodes for enhanced etcd fault tolerance.

**Strengths:**
- etcd tolerates 2 simultaneous failures (instead of 1 with 3 CP)
- Higher API server throughput (5 instances behind load balancer)
- Suitable for large clusters (100+ workers) with high API server load

**Weaknesses:**
- Higher etcd write latency (consensus requires 3/5 instead of 2/3)
- More resources dedicated to control plane
- Rarely necessary — 3 CP is sufficient for most deployments

**When to use:** Large-scale production (100+ workers), multi-tenant platforms, environments with aggressive SLAs requiring higher fault tolerance.

### Multi-Site / Stretched Cluster

Control plane and worker nodes distributed across multiple physical locations, availability zones, or data centers.

**Strengths:**
- Survives entire site failure
- Geographic redundancy for disaster recovery

**Weaknesses:**
- etcd is highly sensitive to network latency — cross-site latency must be under 10ms RTT
- Network partitions can cause split-brain
- Significantly more complex networking (VPN, peering, MTU)
- Not recommended unless network latency requirements can be guaranteed

**When to use:** Only when you have low-latency interconnects between sites (same metro area, direct fiber). For cross-region DR, use separate clusters with federation or Cluster Mesh instead.

---

## Comparison Table

| Topology | Nodes | CP Fault Tolerance | Worker Fault Tolerance | etcd Quorum | Typical Use Case |
|----------|-------|-------------------|----------------------|-------------|-----------------|
| **1 node** | 1 | None | None | 1/1 | Dev, CI/CD, edge |
| **3CP + 3W** | 6 | 1 CP failure | 1-2 W failures | 2/3 | Standard production |
| **3CP + NW** | 3+N | 1 CP failure | Depends on N | 2/3 | Scalable production |
| **5CP + NW** | 5+N | 2 CP failures | Depends on N | 3/5 | Large-scale production |
| **Multi-site** | 3+N per site | Per-site dependent | Per-site dependent | Cross-site | Geo-redundancy |

---

## Node Sizing Guidelines

### Control Plane Nodes

Control plane nodes run etcd, kube-apiserver, kube-controller-manager, and kube-scheduler. Size them based on cluster scale:

| Cluster Size | CPU | RAM | OS Disk | etcd Disk | Notes |
|---|---|---|---|---|---|
| Small (1-10 workers) | 2 cores | 4 GB | 20 GB SSD | Shared with OS | etcd on same disk is acceptable |
| Medium (10-50 workers) | 4 cores | 8 GB | 20 GB SSD | 50 GB dedicated SSD | Separate etcd disk recommended |
| Large (50-100 workers) | 8 cores | 16 GB | 20 GB SSD | 100 GB NVMe | Dedicated NVMe for etcd essential |
| Very large (100+ workers) | 16 cores | 32 GB | 20 GB SSD | 100 GB NVMe | Consider 5 CP nodes |

**etcd disk guidance:**
- etcd performance is dominated by disk write latency
- SSDs are strongly recommended; NVMe for large clusters
- Separate etcd disk from OS disk for medium+ clusters
- Talos places etcd data at `/var/lib/etcd` on the EPHEMERAL partition

### Worker Nodes

Worker node sizing depends entirely on the workloads. General guidelines:

| Workload Profile | CPU | RAM | Disk | Notes |
|---|---|---|---|---|
| General web apps | 4 cores | 8 GB | 50 GB | Moderate CPU, moderate RAM |
| Data services (PostgreSQL, Redis) | 8 cores | 32 GB | 200 GB SSD | High RAM, fast disk |
| Machine learning | 8+ cores + GPU | 64 GB | 500 GB NVMe | GPU passthrough via extensions |
| Edge / IoT | 2 cores | 4 GB | 20 GB | Minimal footprint |

**Talos overhead per node:** ~300MB RAM, ~1 GB disk for the OS. The rest is available for Kubernetes workloads.

---

## Recommendation by Use Case

| Use Case | Topology | Node Sizing | Notes |
|----------|----------|-------------|-------|
| **Learning / development** | 1 node | 4 CPU, 8 GB RAM | Single node with `allowSchedulingOnControlPlanes` |
| **Homelab** | 3 CP + 0-3 W | 4 CPU, 8 GB RAM each | 3 CP with workload scheduling, or add small workers |
| **Small production** | 3 CP + 3 W | CP: 4/8GB, W: 4/16GB | Standard HA, moderate sizing |
| **Medium production** | 3 CP + 5-20 W | CP: 4/8GB, W: 8/32GB | Dedicated etcd disks on CP |
| **Large production** | 3-5 CP + 20-100 W | CP: 8/16GB, W: varies | 5 CP if > 50 workers, heterogeneous workers |
| **Edge site** | 1 node per site | 2/4GB minimum | Managed centrally, local workload execution |
| **Multi-tenant platform** | 5 CP + NW | CP: 16/32GB, W: 8/32GB | 5 CP for API server HA, node pools per tenant |

---

## Migration Path

**Adding worker nodes:** Generate a worker config from the same secrets and apply it to the new node. No disruption to existing cluster.

```bash
# Generate worker config (using existing secrets)
talosctl gen config <cluster-name> <endpoint> \
  --with-secrets <secrets.yaml> \
  --output-types worker \
  --output worker-new.yaml

# Apply to new node
talosctl apply-config --insecure -n <new-node-ip> --file worker-new.yaml
```

**Adding control plane nodes:** Add one at a time. etcd membership is automatic. Ensure you go from 3 to 5 (always odd numbers for etcd quorum).

**Removing control plane nodes:** Remove the etcd member first, then reset the node. Go from 5 to 3, never to 4 or 2.

```bash
# Remove etcd member
talosctl etcd members -n <remaining-cp>
talosctl etcd remove-member <member-id> -n <remaining-cp>

# Reset the removed node
talosctl reset --graceful -n <node-to-remove>
```

**Key takeaway:** Start with the smallest topology that meets your HA requirements and grow from there. It is easier to add nodes than to shrink. Always maintain an odd number of control plane nodes.
