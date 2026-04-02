# Cloud Provider CSI Drivers

Cloud-native storage using provider-managed block storage services. Each requires the corresponding Cloud Controller Manager (CCM) installed first.

## AWS EBS CSI Driver

### Prerequisites
- AWS CCM installed and running
- IAM role with EBS permissions attached to nodes

### Installation

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT:role/ebs-csi-role
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## GCP Persistent Disk CSI Driver

### Prerequisites
- GCP CCM installed and running
- Service account with Compute Engine permissions

### Installation

```bash
kubectl apply -k "github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver/deploy/kubernetes/overlays/stable/?ref=v1.14.0"
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## Azure Disk CSI Driver

### Prerequisites
- Azure CCM installed and running
- Managed identity with Disk Contributor role

### Installation

```bash
helm repo add azuredisk-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/charts
helm repo update

helm install azuredisk-csi-driver azuredisk-csi-driver/azuredisk-csi-driver \
  --namespace kube-system \
  --set cloud=AzurePublicCloud
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingMode: ReadOnly
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## Common Notes

- **WaitForFirstConsumer**: All cloud CSI drivers should use this volume binding mode to ensure the disk is created in the same availability zone as the pod.
- **Encryption**: Enable at-rest encryption via provider parameters. AWS: `encrypted: "true"`. GCP and Azure encrypt by default.
- **Volume expansion**: All cloud CSI drivers support online volume expansion.
- **Snapshots**: All support VolumeSnapshot via the CSI snapshot controller. Install the snapshot CRDs and controller separately if needed.
- **Cost**: Cloud block storage is billed per GB provisioned, plus IOPS for provisioned-IOPS tiers.
