# Decision Guide — CSI Selection

## Context

The Container Storage Interface (CSI) provides persistent storage for Kubernetes workloads. Unlike CNI, CSI selection is a **post-bootstrap decision** that can be changed or augmented at any time. You can run multiple CSI providers simultaneously.

**When to decide:** After cluster bootstrap, before deploying stateful workloads.

**Can it be changed later?** Yes. You can add, remove, or replace CSI providers. However, migrating data between storage backends requires manual effort (backup/restore or volume cloning).

**Talos-specific constraint:** Talos is immutable and has no package manager. CSI drivers that require host-level tools (iSCSI initiator, NFS client, LVM) need those tools installed as **Talos system extensions** in the machine config before they will work.

---

## Options

### Rook-Ceph

Distributed storage system that runs Ceph inside Kubernetes. Provides block (RBD), filesystem (CephFS), and object (S3-compatible) storage.

**Strengths:**
- True distributed storage with configurable replication (2x or 3x)
- Block, filesystem, and object storage from one system
- Self-healing — automatically rebalances when nodes join/leave
- Snapshot and clone support
- Production-proven at scale

**Weaknesses:**
- High resource consumption (~2GB RAM, 1 CPU per OSD pod, plus monitors)
- Requires minimum 3 nodes for production (1 OSD per node minimum)
- Complex to operate — Ceph has a steep learning curve
- Recovery from failures can be slow on spinning disks
- Needs dedicated raw disks or partitions (not shared with OS)

**Talos extensions required:** None (Ceph runs entirely in pods).

**Minimum viable config:**
- 3 nodes, each with at least 1 unused raw disk
- 2GB RAM + 1 CPU per OSD (per disk)
- 1GB RAM for each MON pod (3 MON pods for HA)

### Longhorn

Distributed block storage built for Kubernetes. Simpler than Ceph, with a built-in UI.

**Strengths:**
- Simple installation and operation
- Built-in backup to S3-compatible storage
- Built-in UI dashboard
- Per-volume replica count (1-3)
- Snapshot and backup scheduling
- Lower operational complexity than Ceph

**Weaknesses:**
- Block storage only — no filesystem or object storage
- Requires iSCSI tools on every node (Talos extension required)
- Performance lower than Ceph RBD in benchmarks
- Not suitable for very large clusters (100+ nodes)
- Rebuilding replicas is slower than Ceph rebalancing

**Talos extensions required:**

```yaml
machine:
  install:
    extensions:
      - image: ghcr.io/siderolabs/iscsi-tools:v0.1.6
      - image: ghcr.io/siderolabs/util-linux-tools:v2.40.2
```

### OpenEBS (Mayastor)

High-performance distributed block storage using NVMe-over-Fabrics.

**Strengths:**
- NVMe-oF protocol — high IOPS, low latency
- Synchronous replication
- Simple architecture (no MON/OSD separation like Ceph)
- Growing community and CNCF sandbox project

**Weaknesses:**
- Requires NVMe-capable disks for best performance
- Block storage only
- Smaller community than Ceph or Longhorn
- Less mature backup/restore story
- Requires iSCSI tools extension on Talos

**Talos extensions required:**

```yaml
machine:
  install:
    extensions:
      - image: ghcr.io/siderolabs/iscsi-tools:v0.1.6
```

### Local Path Provisioner

Simplest possible storage — uses local directories on the node filesystem.

**Strengths:**
- Zero dependencies, zero configuration
- Lowest possible latency (direct disk access)
- Minimal resource overhead
- Perfect for single-node clusters and development

**Weaknesses:**
- No replication — data exists on one node only
- Pod must run on the node where data is stored (node affinity)
- No snapshots, no cloning
- Node failure = data loss
- Not suitable for production stateful workloads

**Talos extensions required:** None.

### NFS Subdir External Provisioner

Uses an external NFS server for persistent volumes. Each PVC gets a subdirectory on the NFS share.

**Strengths:**
- Leverages existing NFS infrastructure
- Simple to set up — only needs an NFS server IP and export path
- Shared storage — multiple pods can access the same data (ReadWriteMany)
- No disks required on Kubernetes nodes
- Backup handled at the NFS server level

**Weaknesses:**
- Requires an external NFS server (not self-contained)
- Performance limited by network and NFS server
- NFS server is a single point of failure (unless HA NFS)
- No block storage — filesystem only
- No snapshots from the CSI layer

**Talos extensions required:**

```yaml
machine:
  install:
    extensions:
      - image: ghcr.io/siderolabs/nfs-client:v0.1.0
```

### Cloud CSI Drivers (AWS EBS, GCP PD, Azure Disk)

Native cloud provider block storage attached to VMs.

**Strengths:**
- Fully managed by the cloud provider
- Snapshots, encryption, and resizing built in
- High availability within availability zones
- No additional nodes or disks to manage
- Seamless integration with cloud IAM

**Weaknesses:**
- Only available on that specific cloud platform
- Cost per GB higher than self-managed storage
- Cross-AZ access not possible (EBS, Azure Disk) or expensive
- Vendor lock-in
- Attach/detach latency (10-30 seconds)

**Talos extensions required:** None (cloud CSI drivers run as pods).

---

## Comparison Table

| Feature | Rook-Ceph | Longhorn | OpenEBS Mayastor | Local Path | NFS | Cloud CSI |
|---------|-----------|----------|------------------|------------|-----|-----------|
| **Storage type** | Block, FS, Object | Block | Block | Local FS | NFS (FS) | Block |
| **Replication** | 2x-3x configurable | 1x-3x configurable | 2x-3x | None | Server-side | Cloud-managed |
| **ReadWriteMany** | Yes (CephFS) | Yes (NFS mode) | No | No | Yes | No (EBS/Disk) |
| **Min nodes** | 3 | 3 (for HA) | 3 (for HA) | 1 | 0 (external) | 1 |
| **Dedicated disks** | Required | Recommended | Required | No | No | N/A (cloud) |
| **Talos extensions** | None | iscsi-tools, util-linux-tools | iscsi-tools | None | nfs-client | None |
| **RAM per node** | ~2-4GB | ~500MB | ~1GB | ~0 | ~0 | ~100MB |
| **Snapshots** | Yes | Yes | Yes | No | No | Yes |
| **Backup built-in** | No (use Velero) | Yes (S3) | No (use Velero) | No | Server-side | Cloud snapshots |
| **UI dashboard** | Ceph Dashboard | Yes (built-in) | No | No | No | Cloud console |
| **Performance** | High (RBD) | Medium | Very High (NVMe) | Highest (local) | Low-Medium | Medium-High |
| **Operational complexity** | High | Medium | Medium | Very Low | Low | Very Low |
| **Maturity** | Very mature | Mature | Growing | Stable | Stable | Mature |

---

## Recommendation by Use Case

| Use Case | Recommendation | Reasoning |
|----------|---------------|-----------|
| **Production, on-prem, 3+ nodes** | Rook-Ceph | Distributed, replicated, self-healing, battle-tested |
| **Production, cloud** | Cloud CSI + Velero | Leverage cloud-native storage, add Velero for cross-cloud backup |
| **Homelab, 3 nodes** | Longhorn | Simpler than Ceph, built-in UI and S3 backup, sufficient for homelab scale |
| **Homelab, 1 node** | Local Path Provisioner | No replication needed on single node; simplest possible setup |
| **Existing NAS/NFS** | NFS Subdir Provisioner | Leverage existing infrastructure, no additional disks |
| **High-IOPS workloads** | OpenEBS Mayastor or Rook-Ceph (RBD) | NVMe-oF for Mayastor; RBD with NVMe disks for Ceph |
| **Databases (PostgreSQL, MySQL)** | Rook-Ceph (RBD) or Cloud CSI | Block storage with replication; avoid NFS for databases |
| **Shared filesystem (many writers)** | Rook-Ceph (CephFS) or NFS | Only options that support ReadWriteMany reliably |
| **Budget-constrained** | Longhorn or Local Path | No dedicated hardware required beyond existing node disks |
| **Mixed workloads** | Rook-Ceph + Local Path | Ceph for replicated workloads, Local Path for ephemeral/cache |

---

## Migration Path

CSI providers can coexist. The recommended migration approach:

1. **Install the new CSI provider** alongside the existing one
2. **Create a new StorageClass** for the new provider
3. **For each stateful workload:**
   a. Take a backup (Velero, manual, or application-level)
   b. Create a new PVC with the new StorageClass
   c. Copy data (using a temporary pod with both PVCs mounted, or restore from backup)
   d. Update the workload to use the new PVC
   e. Delete the old PVC
4. **Remove the old CSI provider** once all data is migrated

**Tools for migration:**
- [Velero](https://velero.io/) — backup and restore PVs across storage backends
- `kubectl cp` or rsync pods — for manual data copy between PVCs
- PVC cloning (if both old and new CSI support it within the same provider)

**Key takeaway:** CSI is flexible — you can start simple (Local Path) and add distributed storage (Longhorn, Ceph) as your needs grow. Running multiple CSI providers is normal and expected.
