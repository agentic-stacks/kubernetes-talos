# NFS CSI Driver

The NFS CSI driver provides persistent storage backed by an external NFS server.

## When to Use

- Existing NFS infrastructure you want to leverage
- ReadWriteMany access mode needed (multiple pods reading/writing)
- Simple shared storage without cluster-internal storage management

## Prerequisites

- External NFS server accessible from all Talos nodes
- **nfs-client system extension** — required for NFS mount support on Talos

Include in Image Factory schematic:

```yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/nfs-mount
```

## Installation

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update

helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system
```

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.example.com
  share: /exports/kubernetes
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4.1
  - hard
```

## Static PV Example

For existing NFS exports:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-data
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs-server.example.com
    path: /exports/data
  mountOptions:
    - nfsvers=4.1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-data
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  volumeName: nfs-data
  resources:
    requests:
      storage: 100Gi
```

## Operational Notes

- **ReadWriteMany**: NFS is the simplest way to get RWX volumes.
- **Performance**: Limited by NFS server and network. Not suitable for high-IOPS workloads.
- **Reliability**: Depends entirely on the external NFS server. No Kubernetes-managed replication.
- **Permissions**: Ensure NFS exports allow access from Talos node IPs with appropriate `uid`/`gid` mappings.
