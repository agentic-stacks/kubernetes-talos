# Microsoft Azure

## Prerequisites

- Azure subscription
- `az` CLI authenticated
- Resource group created
- Virtual network and subnet
- Managed identity or service principal for Azure CCM

## Image Provisioning

Upload the Talos Azure VHD:

```bash
# Download Azure image
wget https://factory.talos.dev/image/<schematic-id>/v1.9.0/azure-amd64.vhd.xz
xz -d azure-amd64.vhd.xz

# Create storage account and container
az storage account create -n talosstorage -g myResourceGroup --sku Standard_LRS
az storage container create -n images --account-name talosstorage

# Upload VHD
az storage blob upload \
  --account-name talosstorage \
  --container-name images \
  --name talos-v1.9.0.vhd \
  --file azure-amd64.vhd \
  --type page

# Create image
az image create \
  --resource-group myResourceGroup \
  --name talos-v1-9-0 \
  --os-type Linux \
  --source https://talosstorage.blob.core.windows.net/images/talos-v1.9.0.vhd
```

## Network Requirements

**Network Security Group (NSG):**

```bash
# Create NSG
az network nsg create -g myResourceGroup -n talos-nsg

# Talos API
az network nsg rule create -g myResourceGroup --nsg-name talos-nsg \
  -n talos-api --priority 100 --access Allow --protocol Tcp \
  --destination-port-ranges 50000 50001

# K8s API
az network nsg rule create -g myResourceGroup --nsg-name talos-nsg \
  -n k8s-api --priority 110 --access Allow --protocol Tcp \
  --destination-port-ranges 6443

# etcd (CP only — use ASG for source)
az network nsg rule create -g myResourceGroup --nsg-name talos-nsg \
  -n etcd --priority 120 --access Allow --protocol Tcp \
  --destination-port-ranges 2379-2380

# kubelet
az network nsg rule create -g myResourceGroup --nsg-name talos-nsg \
  -n kubelet --priority 130 --access Allow --protocol Tcp \
  --destination-port-ranges 10250
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
    - <lb-ip-or-dns>
cluster:
  controlPlane:
    endpoint: https://<lb-ip-or-dns>:6443
  externalCloudProvider:
    enabled: true
```

Install Azure CCM separately after bootstrap:

```bash
helm repo add cloud-provider-azure https://raw.githubusercontent.com/kubernetes-sigs/cloud-provider-azure/master/helm/repo
helm install azure-ccm cloud-provider-azure/cloud-provider-azure \
  --set cloudControllerManager.cloudConfig=/etc/kubernetes/azure.json
```

## Control Plane Endpoint

**Azure Load Balancer — Recommended:**

```bash
# Create public IP
az network public-ip create \
  -g myResourceGroup -n talos-cp-ip --sku Standard

# Create load balancer
az network lb create \
  -g myResourceGroup -n talos-cp-lb \
  --sku Standard \
  --frontend-ip-name cp-frontend \
  --public-ip-address talos-cp-ip \
  --backend-pool-name cp-backend

# Create health probe
az network lb probe create \
  -g myResourceGroup --lb-name talos-cp-lb \
  -n cp-probe --protocol Tcp --port 6443

# Create rule
az network lb rule create \
  -g myResourceGroup --lb-name talos-cp-lb \
  -n cp-rule --protocol Tcp \
  --frontend-port 6443 --backend-port 6443 \
  --frontend-ip-name cp-frontend \
  --backend-pool-name cp-backend \
  --probe-name cp-probe
```

## Provisioning Commands

```bash
# Create control plane VMs
for i in 1 2 3; do
  az vm create \
    --resource-group myResourceGroup \
    --name talos-cp${i} \
    --image talos-v1-9-0 \
    --size Standard_D4s_v3 \
    --os-disk-size-gb 50 \
    --storage-sku Premium_LRS \
    --nsg talos-nsg \
    --vnet-name myVnet \
    --subnet mySubnet \
    --assign-identity \
    --custom-data controlplane.yaml \
    --no-wait
done

# Create worker VMs
for i in 1 2 3; do
  az vm create \
    --resource-group myResourceGroup \
    --name talos-worker${i} \
    --image talos-v1-9-0 \
    --size Standard_D8s_v3 \
    --os-disk-size-gb 100 \
    --storage-sku Premium_LRS \
    --nsg talos-nsg \
    --vnet-name myVnet \
    --subnet mySubnet \
    --assign-identity \
    --custom-data worker.yaml \
    --no-wait
done
```

## Platform-Specific Gotchas

- **No VIP**: Azure does not support gratuitous ARP. Use Azure LB.
- **Managed identity**: Preferred over service principal for Azure CCM. Assign `Contributor` role to the identity.
- **Premium storage**: Use Premium_LRS for control plane (etcd performance).
- **Availability zones**: Distribute control plane nodes across zones with `--zone 1 2 3`.
- **Custom data**: Azure passes machine config via `--custom-data`. The file is delivered as cloud-init user-data.
- **Accelerated networking**: Enable `--accelerated-networking true` for better network performance on supported VM sizes.
