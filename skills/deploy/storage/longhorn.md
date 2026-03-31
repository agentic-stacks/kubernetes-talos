# Longhorn on Talos

Longhorn is a lightweight distributed block storage system built on and for Kubernetes.

## Prerequisites

- **iscsi-tools system extension required** — Longhorn uses iSCSI for block device management
- Minimum 3 nodes for replication
- Dedicated disk or partition for Longhorn data (recommended, not required)

## Talos Machine Config

Include the iSCSI extension in your Image Factory schematic:

```yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/iscsi-tools
```

Or add to machine config:

```yaml
machine:
  install:
    extensions:
      - image: ghcr.io/siderolabs/iscsi-tools:v0.1.4
```

Optionally dedicate a disk for Longhorn:

```yaml
machine:
  disks:
    - device: /dev/sdb
      partitions:
        - mountpoint: /var/lib/longhorn
```

## Installation

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system --create-namespace \
  --set defaultSettings.defaultDataPath=/var/lib/longhorn \
  --set defaultSettings.defaultReplicaCount=3
```

## StorageClass

Longhorn creates a default StorageClass automatically. Customize if needed:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  fromBackup: ""
  dataLocality: disabled
```

## Verification

```bash
# Check Longhorn pods
kubectl -n longhorn-system get pods

# Check nodes recognized by Longhorn
kubectl -n longhorn-system get nodes.longhorn.io

# Access UI via port-forward
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
```

## Backup to S3

Configure backup targets for disaster recovery:

```bash
# Create S3 secret
kubectl -n longhorn-system create secret generic s3-backup-secret \
  --from-literal=AWS_ACCESS_KEY_ID=<key> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<secret> \
  --from-literal=AWS_ENDPOINTS=https://s3.amazonaws.com
```

Set backup target in Longhorn settings: `s3://my-bucket@us-east-1/backups`

## Operational Notes

- **Simple**: Easier to set up than Rook-Ceph, fewer moving parts.
- **Snapshots**: Longhorn supports volume snapshots and recurring snapshot schedules.
- **Read-Write-Many**: Supported via NFS export (built into Longhorn).
- **Performance**: Moderate — suitable for general workloads, not ideal for high-IOPS databases. Use dedicated disks for better performance.
- **Upgrade**: `helm upgrade longhorn longhorn/longhorn -n longhorn-system`
