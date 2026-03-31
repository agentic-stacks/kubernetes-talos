# Rook-Ceph on Talos

Rook deploys and manages Ceph clusters on Kubernetes. It provides block (RBD), filesystem (CephFS), and object (S3-compatible) storage.

## Prerequisites

- Minimum 3 nodes with dedicated raw disks (not the Talos install disk)
- At least 1 raw disk per node for OSD (Object Storage Daemon)
- No system extensions required — Ceph runs entirely in containers
- 4 GB RAM per OSD recommended

## Talos Machine Config

Dedicate disks for Ceph in the machine config. Do NOT partition them — Rook needs raw block devices:

```yaml
machine:
  disks: []
  # Leave additional disks unconfigured — Rook will discover and use them
  # Do NOT add them to machine.disks unless you want Talos to partition them
```

Alternatively, explicitly list disks for Talos to ignore:

```yaml
machine:
  disks:
    - device: /dev/sdb
      partitions: []    # Empty — signals Talos to not touch this disk
```

Verify available disks on a running node:

```bash
talosctl -n <IP> disks
# Look for disks that are not the install disk and have no partitions
```

## Installation

```bash
# Add the Rook Helm repo
helm repo add rook-release https://charts.rook.io/release
helm repo update

# Install the Rook operator
helm install rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph --create-namespace \
  --set csi.cephFSPluginVolume=null \
  --set csi.cephFSPluginVolumeMount=null

# Wait for operator to be ready
kubectl -n rook-ceph wait deployment rook-ceph-operator --for=condition=available --timeout=300s
```

## Create the Ceph Cluster

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  dataDirHostPath: /var/lib/rook
  cephVersion:
    image: quay.io/ceph/ceph:v18.2
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 2
    modules:
      - name: pg_autoscaler
        enabled: true
  dashboard:
    enabled: true
  storage:
    useAllNodes: true
    useAllDevices: true
    # Or be explicit:
    # nodes:
    #   - name: worker1
    #     devices:
    #       - name: sdb
    #   - name: worker2
    #     devices:
    #       - name: sdb
```

```bash
kubectl apply -f ceph-cluster.yaml

# Monitor progress
kubectl -n rook-ceph get cephcluster -w
# Wait for HEALTH_OK status (can take 5-10 minutes)
```

## StorageClass

**Block storage (RBD):**

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

**Filesystem storage (CephFS):**

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: cephfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-filesystem
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: cephfs
  pool: cephfs-data0
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Verification

```bash
# Check cluster health
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status
# Look for: health HEALTH_OK

# Check OSDs
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd tree

# Test a PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: ceph-block
  resources:
    requests:
      storage: 1Gi
EOF
kubectl get pvc test-pvc  # Should be Bound
```

## Operational Notes

- **Dashboard**: Access via port-forward: `kubectl -n rook-ceph port-forward svc/rook-ceph-mgr-dashboard 7000:7000`
- **Monitoring**: Rook exports Prometheus metrics. Create ServiceMonitors for integration with kube-prometheus-stack.
- **Scaling**: Add more OSDs by adding disks to nodes or adding new nodes. Ceph rebalances automatically.
- **Upgrades**: Follow the Rook upgrade guide — upgrade operator first, then CephCluster CR.
