# Deploy — Storage

Persistent storage for Talos Linux clusters. Talos is an immutable OS — it has no package manager and no writable host filesystem for storage daemons. Every CSI solution must account for this constraint. This skill covers CSI driver selection, Talos-specific prerequisites, and installation guides for each supported option.

---

## 1. Talos Storage Model

Talos Linux enforces a read-only root filesystem. This changes how storage works compared to conventional Linux:

- **No host-path manipulation** — storage daemons cannot write to arbitrary host directories during runtime. Paths under `/var/lib` are writable (they live on the state partition), and `/mnt` is available for CSI staging targets.
- **No package manager** — kernel modules (iSCSI initiator, ZFS, NFS client) cannot be installed at runtime. They must be baked into the Talos image as system extensions.
- **Disk management via machine config** — raw block devices are configured with `machine.disks` in the Talos machine config, not with `fdisk` or `parted` at the shell. Talos applies the partition layout on boot.
- **No SSH, no shell** — operational tasks must go through `talosctl`, the Talos API, or Kubernetes-native tooling (kubectl, Helm).

### Writable Paths Available to CSI Drivers

| Path | Partition | Purpose |
|---|---|---|
| `/var/lib/kubelet` | STATE | Kubelet data, plugin sockets |
| `/var/lib/longhorn` | STATE (configurable) | Longhorn replica storage |
| `/var/local-path-provisioner` | STATE (configurable) | local-path-provisioner data |
| `/mnt` | EPHEMERAL or data disk | CSI staging and publish targets |

---

## 2. System Extensions for Storage

Several CSI drivers require kernel-level support that Talos does not include in the base image. Add extensions via the **Image Factory** schematic before provisioning nodes. Existing nodes must be upgraded to a new image that includes the required extensions.

### Available Storage Extensions

| Extension | Required By | Purpose |
|---|---|---|
| `siderolabs/iscsi-tools` | Longhorn, OpenEBS (Jiva) | iSCSI initiator (`iscsid`, `iscsiadm`) |
| `siderolabs/zfs` | ZFS-backed storage | ZFS kernel module and userspace tools |
| `siderolabs/nfs-client` | NFS CSI driver | `nfs`, `nfsd`, and `sunrpc` kernel modules |

### Adding Extensions via Image Factory Schematic

Create a schematic file and submit it to `factory.talos.dev`:

```yaml
# schematic.yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/iscsi-tools
```

```bash
# Submit schematic and receive a schematic ID
curl -X POST --data-binary @schematic.yaml \
  https://factory.talos.dev/schematics

# Response contains the schematic ID, e.g.:
# {"id":"376567988ad370138ad8ee98abb6f8b75f7a84fe33b86c9ed4d3a6b30e1baf47"}

SCHEMATIC_ID="376567988ad370138ad8ee98abb6f8b75f7a84fe33b86c9ed4d3a6b30e1baf47"
TALOS_VERSION="v1.9.5"

# Download the image with the extension baked in
curl -LO "https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/metal-amd64.iso"
```

For combined extensions (e.g., iSCSI + NFS):

```yaml
# schematic-iscsi-nfs.yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/iscsi-tools
      - siderolabs/nfs-client
```

### Upgrading Existing Nodes to Add Extensions

```bash
# Upgrade a running node to an image that includes the new extension
talosctl upgrade \
  --nodes 10.0.0.21 \
  --image "factory.talos.dev/installer/${SCHEMATIC_ID}:${TALOS_VERSION}"

# Verify extensions are active after upgrade
talosctl get extensions --nodes 10.0.0.21
```

---

## 3. Disk Management via machine.disks

Talos configures disk partitioning declaratively in the machine config. For storage drivers that need dedicated block devices (Rook-Ceph OSDs, Longhorn replicas on a separate disk), define the disk layout before first boot or apply a patch to running nodes.

```yaml
# machine config patch — dedicated data disk at /dev/sdb
machine:
  disks:
    - device: /dev/sdb
      partitions:
        - mountpoint: /var/mnt/data
          size: 0  # 0 = use remaining space
```

Rook-Ceph and Mayastor can also operate on raw unpartitioned disks. In that case, leave `machine.disks` unset for those devices and let the CSI driver claim them directly.

```bash
# Discover disk device names on a node
talosctl disks --nodes 10.0.0.21
```

---

## 4. Decision Matrix

| Feature | Rook-Ceph | Longhorn | OpenEBS | Local Path | NFS | Cloud CSI |
|---|---|---|---|---|---|---|
| Replication | Yes | Yes (2–3) | Yes (Mayastor) | No | No | Provider-managed |
| Min nodes | 3 | 3 | 3 | 1 | 1 (external) | 1 |
| Raw block | Yes | Yes | Yes | No | No | Yes |
| Performance | High | Medium | High (NVMe) | Highest | Low–Med | Medium |
| Complexity | High | Low–Med | Medium | Minimal | Low | Low |
| Cloud compat | No (raw disks) | Yes | Yes | Yes | Yes | Native |
| Extensions | None | iscsi-tools | iscsi-tools | None | nfs-client | None |
| ReadWriteMany | CephFS | No | No | No | Yes | Provider-dependent |
| Snapshots | Yes | Yes | Yes | No | No | Yes |

**Decision rules:**

- Production on-prem with 3+ nodes and raw disks → **rook-ceph** for the full Ceph feature set
- Production on-prem with 3+ nodes, wants simplicity → **longhorn** for easy day-2 operations
- NVMe-heavy clusters needing maximum IOPS → **openebs** Mayastor engine
- Dev clusters, single-node, or CI environments → **local-path** for zero overhead
- Workloads that need ReadWriteMany against an existing NAS → **nfs**
- Cloud-provisioned clusters → **cloud-storage** (EBS, GCP PD, or Azure Disk)

---

## 5. Disk Encryption

Talos supports LUKS2 full-disk encryption configured in the machine config. This encrypts the STATE and EPHEMERAL partitions and can extend to data disks managed by Talos.

```yaml
machine:
  systemDiskEncryption:
    state:
      provider: luks2
      keys:
        - nodeID: {}
          slot: 0
    ephemeral:
      provider: luks2
      keys:
        - nodeID: {}
          slot: 0
```

Key providers:

| Provider | Description | Use Case |
|---|---|---|
| `nodeID` | Derives key from the unique Talos node identity | Default, works without TPM |
| `tpm` | Seals key to the TPM; decryption requires correct PCR state | Bare metal with TPM 2.0 |
| `static` | Fixed passphrase (base64-encoded) | Testing only — not for production |

TPM sealing binds decryption to the specific hardware and boot state. If the firmware or boot configuration changes, the key cannot be unsealed and the node will not boot without manual intervention.

```yaml
# TPM-sealed encryption
machine:
  systemDiskEncryption:
    state:
      provider: luks2
      keys:
        - tpm: {}
          slot: 0
```

---

## 6. Storage Skill Index

| Option | Guide | Best For |
|---|---|---|
| Rook-Ceph | [rook-ceph.md](./rook-ceph.md) | Production on-prem, full Ceph feature set |
| Longhorn | [longhorn.md](./longhorn.md) | Production on-prem, operational simplicity |
| OpenEBS | [openebs.md](./openebs.md) | NVMe-heavy clusters, high IOPS |
| Local Path | [local-path.md](./local-path.md) | Dev/single-node, minimal overhead |
| NFS | [nfs.md](./nfs.md) | External NAS, ReadWriteMany workloads |
| Cloud Storage | [cloud-storage.md](./cloud-storage.md) | AWS EBS, GCP PD, Azure Disk |

---

## Next Step

After deploying storage, configure workload-specific storage classes and validate PVC provisioning:

```
deploy/storage → platform/applications
```
