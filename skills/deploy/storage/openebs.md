# OpenEBS (Mayastor) on Talos

OpenEBS Mayastor provides high-performance distributed block storage optimized for NVMe drives.

## Prerequisites

- **iscsi-tools system extension required**
- Minimum 3 nodes with NVMe drives for optimal performance (works with any block device)
- 2 GB HugePages per Mayastor node (for SPDK/NVMe-oF)

## Talos Machine Config

Include the iSCSI extension:

```yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/iscsi-tools
```

Configure HugePages for Mayastor:

```yaml
machine:
  sysctls:
    vm.nr_hugepages: "1024"
  kubelet:
    extraArgs:
      system-reserved: "memory=1500Mi"
```

## Installation

```bash
helm repo add openebs https://openebs.github.io/openebs
helm repo update

helm install openebs openebs/openebs \
  --namespace openebs --create-namespace \
  --set mayastor.enabled=true \
  --set engines.replicated.mayastor.enabled=true
```

## DiskPool Configuration

After installation, create DiskPools pointing to available disks:

```yaml
apiVersion: openebs.io/v1beta2
kind: DiskPool
metadata:
  name: pool-worker1
  namespace: openebs
spec:
  node: worker1
  disks:
    - uring:///dev/nvme1n1
---
apiVersion: openebs.io/v1beta2
kind: DiskPool
metadata:
  name: pool-worker2
  namespace: openebs
spec:
  node: worker2
  disks:
    - uring:///dev/nvme1n1
```

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mayastor-3
provisioner: io.openebs.csi-mayastor
parameters:
  protocol: nvmf
  repl: "3"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

## Verification

```bash
kubectl -n openebs get pods
kubectl -n openebs get diskpool
kubectl -n openebs get msv   # Mayastor volumes
```

## Operational Notes

- **Best for NVMe**: Mayastor uses SPDK for kernel-bypass I/O, delivering near-native NVMe performance.
- **HugePages**: Required for SPDK. The 1024 pages (2 MB each) = 2 GB reserved for Mayastor.
- **Resource overhead**: Higher than Longhorn due to SPDK/HugePages requirements.
- **Alternative: OpenEBS Local PV**: For simple local storage without replication, use `openebs-hostpath` StorageClass (no extensions needed).
