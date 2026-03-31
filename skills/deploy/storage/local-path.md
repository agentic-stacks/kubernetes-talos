# Local Path Provisioner

The Rancher Local Path Provisioner provides the simplest possible dynamic storage on Kubernetes using local node storage.

## When to Use

- Development and testing clusters
- Single-node clusters
- Workloads that don't need replication
- Quick setup without storage infrastructure

## Prerequisites

- No system extensions required
- Works on any Talos deployment

## Installation

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.28/deploy/local-path-storage.yaml
```

Or via Helm:

```bash
helm repo add local-path-provisioner https://charts.containeroo.ch
helm install local-path-provisioner local-path-provisioner/local-path-provisioner \
  --namespace local-path-storage --create-namespace \
  --set storageClass.defaultClass=true
```

## StorageClass

The installation creates a StorageClass automatically:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

`WaitForFirstConsumer` ensures the PV is created on the node where the pod is scheduled.

## Verification

```bash
kubectl -n local-path-storage get pods
kubectl get sc local-path

# Test
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-local
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
EOF
```

## Limitations

- **No replication**: Data exists only on the node where the pod runs. Node failure = data loss.
- **No snapshots**: No snapshot or backup capabilities.
- **No volume expansion**: Cannot resize volumes after creation.
- **Node affinity**: Pods using local-path PVCs are tied to the node where the PV was created.
- **Ephemeral partition**: Data is stored on Talos's ephemeral partition (`/var`), which is wiped on Talos upgrades unless `--preserve` is used.
