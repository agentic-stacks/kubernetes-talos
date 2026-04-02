# Google Cloud Platform (GCP)

## Prerequisites

- GCP project with Compute Engine API enabled
- `gcloud` CLI authenticated and configured
- VPC network and subnet
- Service account for nodes (for GCP CCM)

## Image Provisioning

Upload the Talos GCP image:

```bash
# Download GCP image
wget https://factory.talos.dev/image/<schematic-id>/v1.9.0/gcp-amd64.raw.tar.gz

# Upload to GCS
gsutil cp gcp-amd64.raw.tar.gz gs://my-bucket/talos/

# Create image
gcloud compute images create talos-v1-9-0 \
  --source-uri=gs://my-bucket/talos/gcp-amd64.raw.tar.gz \
  --guest-os-features=VIRTIO_SCSI_MULTIQUEUE
```

## Network Requirements

**Firewall rules:**

```bash
# Talos API (admin access)
gcloud compute firewall-rules create talos-api \
  --network=default \
  --allow=tcp:50000 \
  --source-ranges=<admin-cidr> \
  --target-tags=talos

# Internal cluster communication
gcloud compute firewall-rules create talos-internal \
  --network=default \
  --allow=tcp:50000,tcp:50001,tcp:6443,tcp:2379-2380,tcp:10250 \
  --source-tags=talos \
  --target-tags=talos
```

## Machine Config Specifics

```yaml
machine:
  install:
    disk: /dev/sda
    image: ghcr.io/siderolabs/installer:v1.9.0
  kubelet:
    extraArgs:
      cloud-provider: external
  certSANs:
    - <lb-ip-address>
cluster:
  controlPlane:
    endpoint: https://<lb-ip-address>:6443
  externalCloudProvider:
    enabled: true
    manifests:
      - https://raw.githubusercontent.com/kubernetes/cloud-provider-gcp/master/deploy/cloud-controller-manager/latest.yaml
```

## Control Plane Endpoint

**Internal TCP Load Balancer — Recommended:**

```bash
# Create instance group for CP nodes
gcloud compute instance-groups unmanaged create talos-cp-ig \
  --zone=us-central1-a

gcloud compute instance-groups unmanaged add-instances talos-cp-ig \
  --zone=us-central1-a \
  --instances=talos-cp1,talos-cp2,talos-cp3

# Create health check
gcloud compute health-checks create tcp talos-cp-health \
  --port=6443

# Create backend service
gcloud compute backend-services create talos-cp-backend \
  --load-balancing-scheme=INTERNAL \
  --protocol=TCP \
  --health-checks=talos-cp-health \
  --region=us-central1

gcloud compute backend-services add-backend talos-cp-backend \
  --instance-group=talos-cp-ig \
  --instance-group-zone=us-central1-a \
  --region=us-central1

# Create forwarding rule
gcloud compute forwarding-rules create talos-cp-fwd \
  --load-balancing-scheme=INTERNAL \
  --backend-service=talos-cp-backend \
  --ports=6443 \
  --region=us-central1 \
  --subnet=default
```

## Provisioning Commands

```bash
# Create control plane instances
for i in 1 2 3; do
  gcloud compute instances create talos-cp${i} \
    --image=talos-v1-9-0 \
    --machine-type=e2-standard-4 \
    --boot-disk-size=50GB \
    --boot-disk-type=pd-ssd \
    --zone=us-central1-a \
    --tags=talos,talos-cp \
    --can-ip-forward \
    --service-account=talos-node@project.iam.gserviceaccount.com \
    --scopes=cloud-platform
done

# Create worker instances
for i in 1 2 3; do
  gcloud compute instances create talos-worker${i} \
    --image=talos-v1-9-0 \
    --machine-type=e2-standard-8 \
    --boot-disk-size=100GB \
    --boot-disk-type=pd-ssd \
    --zone=us-central1-a \
    --tags=talos,talos-worker \
    --can-ip-forward \
    --service-account=talos-node@project.iam.gserviceaccount.com \
    --scopes=cloud-platform
done
```

**Pass machine config via metadata:**

```bash
gcloud compute instances create talos-cp1 \
  --metadata-from-file=user-data=controlplane.yaml \
  ...
```

## Platform-Specific Gotchas

- **No VIP**: GCP does not support gratuitous ARP. Use Internal TCP LB for control plane endpoint.
- **IP forwarding**: Enable `--can-ip-forward` on all nodes (required for pod networking).
- **Service account**: Create a dedicated SA with `roles/compute.admin` and `roles/compute.loadBalancerAdmin` for the CCM.
- **Disk type**: Use `pd-ssd` for control plane (etcd performance). `pd-balanced` acceptable for workers.
- **Preemptible/Spot**: Workers can use spot instances for cost savings. Never use spot for control plane.
- **Cilium on GCP**: When using Cilium with kube-proxy replacement, set `--set bpf.lbExternalClusterIP=true` for GCP internal LB compatibility.
