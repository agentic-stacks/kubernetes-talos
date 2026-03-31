# DigitalOcean

## Prerequisites

- DigitalOcean account
- `doctl` CLI installed and authenticated
- SSH key added to account

## Image Provisioning

Upload the Talos image as a custom image:

```bash
# Get the DigitalOcean raw image URL from Image Factory
# Upload directly via API
doctl compute image create talos-v1-9-0 \
  --image-url https://factory.talos.dev/image/<schematic-id>/v1.9.0/digital-ocean-amd64.raw.gz \
  --region nyc1 \
  --image-description "Talos Linux v1.9.0"
```

Wait for the image to become available:

```bash
doctl compute image list --public=false
```

## Network Requirements

- DigitalOcean VPC for private networking (recommended)
- Cloud Firewall rules for public access control

```bash
doctl compute firewall create \
  --name talos-fw \
  --inbound-rules "protocol:tcp,ports:50000,address:0.0.0.0/0 protocol:tcp,ports:6443,address:0.0.0.0/0" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0 protocol:udp,ports:all,address:0.0.0.0/0"
```

## Machine Config Specifics

```yaml
machine:
  install:
    disk: /dev/vda    # DigitalOcean uses virtio
    image: ghcr.io/siderolabs/installer:v1.9.0
  kubelet:
    extraArgs:
      cloud-provider: external
  certSANs:
    - <lb-ip>
cluster:
  controlPlane:
    endpoint: https://<lb-ip>:6443
  externalCloudProvider:
    enabled: true
```

Install the DO CCM after bootstrap:

```bash
kubectl create secret generic digitalocean --from-literal=access-token=<DO_TOKEN> -n kube-system
kubectl apply -f https://raw.githubusercontent.com/digitalocean/digitalocean-cloud-controller-manager/master/releases/latest/cloud-controller-manager.yaml
```

## Control Plane Endpoint

**DigitalOcean Load Balancer:**

```bash
# Create load balancer (TCP passthrough for K8s API)
doctl compute load-balancer create \
  --name talos-cp-lb \
  --region nyc1 \
  --forwarding-rules "entry_protocol:tcp,entry_port:6443,target_protocol:tcp,target_port:6443" \
  --health-check "protocol:tcp,port:6443,check_interval_seconds:10,response_timeout_seconds:5,healthy_threshold:3,unhealthy_threshold:3" \
  --droplet-ids <cp1-id>,<cp2-id>,<cp3-id>
```

## Provisioning Commands

```bash
# Create control plane droplets
for i in 1 2 3; do
  doctl compute droplet create talos-cp${i} \
    --image <custom-image-id> \
    --size s-4vcpu-8gb \
    --region nyc1 \
    --vpc-uuid <vpc-id> \
    --ssh-keys <key-fingerprint> \
    --user-data-file controlplane.yaml \
    --tag-names talos,talos-cp \
    --wait
done

# Create worker droplets
for i in 1 2 3; do
  doctl compute droplet create talos-worker${i} \
    --image <custom-image-id> \
    --size s-8vcpu-16gb \
    --region nyc1 \
    --vpc-uuid <vpc-id> \
    --ssh-keys <key-fingerprint> \
    --user-data-file worker.yaml \
    --tag-names talos,talos-worker \
    --wait
done
```

## Platform-Specific Gotchas

- **Disk path**: DigitalOcean uses virtio disks — `/dev/vda` in machine config.
- **No VIP**: Use DigitalOcean Load Balancer for control plane endpoint.
- **VPC**: Create a VPC for private networking. Use private IPs for etcd communication.
- **Block storage**: DO volumes can be attached for persistent storage. Use the DO CSI driver.
- **Droplet sizes**: `s-4vcpu-8gb` minimum for control plane. Premium CPU-optimized droplets recommended for production.
- **Regions**: Ensure all nodes are in the same region for low-latency etcd.
