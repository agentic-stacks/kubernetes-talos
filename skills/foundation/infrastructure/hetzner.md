# Hetzner Cloud

## Prerequisites

- Hetzner Cloud account
- `hcloud` CLI installed and configured with API token
- SSH key uploaded to Hetzner Cloud (required for server creation, not used by Talos)

## Image Provisioning

Upload a Talos image as a custom snapshot:

```bash
# Create a temporary rescue server to write the image
hcloud server create --name talos-imager --type cx22 --image ubuntu-24.04 --ssh-key mykey

# SSH in and write the Talos image
ssh root@<server-ip>
wget https://factory.talos.dev/image/<schematic-id>/v1.9.0/hcloud-amd64.raw.xz
xz -d hcloud-amd64.raw.xz
dd if=hcloud-amd64.raw of=/dev/sda bs=4M status=progress
sync

# Power off and create snapshot
hcloud server shutdown talos-imager
hcloud server create-image --type snapshot --description "Talos v1.9.0" talos-imager

# Note the snapshot ID, then clean up
hcloud server delete talos-imager
```

Alternatively, use Talos Image Factory with the `hcloud` platform.

## Network Requirements

- Hetzner Cloud firewall or server-level iptables (Talos manages its own)
- Private network recommended for inter-node communication

```bash
# Create private network
hcloud network create --name talos-net --ip-range 10.0.0.0/8
hcloud network add-subnet talos-net --type server --network-zone eu-central --ip-range 10.0.1.0/24
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
    - <lb-ip>
cluster:
  controlPlane:
    endpoint: https://<lb-ip>:6443
  externalCloudProvider:
    enabled: true
```

Install the Hetzner CCM after bootstrap:

```bash
helm repo add hcloud https://charts.hetzner.cloud
helm install hccm hcloud/hcloud-cloud-controller-manager \
  -n kube-system \
  --set env.HCLOUD_TOKEN.valueFrom.secretKeyRef.name=hcloud \
  --set env.HCLOUD_TOKEN.valueFrom.secretKeyRef.key=token \
  --set networking.enabled=true \
  --set networking.clusterCIDR=10.244.0.0/16
```

## Control Plane Endpoint

**Hetzner Load Balancer — Recommended:**

```bash
hcloud load-balancer create --name talos-cp-lb --type lb11 --location fsn1

hcloud load-balancer add-service talos-cp-lb \
  --protocol tcp --listen-port 6443 --destination-port 6443

hcloud load-balancer add-target talos-cp-lb --server talos-cp1
hcloud load-balancer add-target talos-cp-lb --server talos-cp2
hcloud load-balancer add-target talos-cp-lb --server talos-cp3
```

**Alternative: Floating IP** — assign to one CP node, failover manually or with a controller.

## Provisioning Commands

```bash
# Create control plane servers
for i in 1 2 3; do
  hcloud server create \
    --name talos-cp${i} \
    --type cpx31 \
    --image <snapshot-id> \
    --location fsn1 \
    --ssh-key mykey \
    --network talos-net \
    --user-data-from-file controlplane.yaml
done

# Create worker servers
for i in 1 2 3; do
  hcloud server create \
    --name talos-worker${i} \
    --type cpx41 \
    --image <snapshot-id> \
    --location fsn1 \
    --ssh-key mykey \
    --network talos-net \
    --user-data-from-file worker.yaml
done
```

## Platform-Specific Gotchas

- **No VIP on public network**: Hetzner does not allow gratuitous ARP on public IPs. Use Hetzner LB or floating IP. VIP works on private networks.
- **Dedicated servers**: For bare metal Hetzner dedicated servers, use the bare-metal guide with PXE/rescue boot instead.
- **Private network**: Strongly recommended. Use the private network IP for etcd and inter-node communication via `machine.network.interfaces` and `cluster.etcd.advertisedSubnets`.
- **Server types**: `cpx31` (4 vCPU, 8 GB) is a good minimum for control plane. `cpx41` or `ccx` series for workers.
- **Storage volumes**: Hetzner Cloud Volumes can be attached for persistent storage. Use the Hetzner CSI driver.
