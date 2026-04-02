# Libvirt (QEMU/KVM)

## Prerequisites

- Linux host with KVM support (`kvm-ok` or `lsmod | grep kvm`)
- `libvirt`, `qemu-kvm`, and `virt-install` installed
- Bridge networking configured (e.g., `br0`) or default NAT network
- Talos qcow2 image downloaded
- `virsh` CLI access

## Image Provisioning

Download the metal qcow2 image:

```bash
wget https://factory.talos.dev/image/<schematic-id>/v1.9.0/metal-amd64.qcow2
```

Or use the nocloud variant:

```bash
wget https://factory.talos.dev/image/<schematic-id>/v1.9.0/nocloud-amd64.qcow2
```

## Network Requirements

- Bridge networking recommended for production (VMs on same L2 as host)
- NAT networking works for development but prevents external access to VIP
- Talos VIP requires bridge mode (gratuitous ARP)
- Firewall: allow TCP 50000, 50001, 6443, 2379-2380, 10250

## Machine Config Specifics

```yaml
machine:
  install:
    disk: /dev/vda    # virtio disk
    image: ghcr.io/siderolabs/installer:v1.9.0
    extensions:
      - image: ghcr.io/siderolabs/qemu-guest-agent:v1.9.0
```

Note: libvirt/QEMU uses `/dev/vda` for virtio disks, not `/dev/sda`.

## Control Plane Endpoint

**Talos VIP** works on bridge networks:

```yaml
machine:
  network:
    interfaces:
      - interface: eth0
        dhcp: true
        vip:
          ip: 192.168.122.100
```

## Provisioning Commands

**Create VMs with virt-install:**

```bash
# Copy base image for each node
for node in cp1 cp2 cp3 worker1 worker2; do
  cp metal-amd64.qcow2 /var/lib/libvirt/images/talos-${node}.qcow2
  qemu-img resize /var/lib/libvirt/images/talos-${node}.qcow2 40G
done

# Create control plane VM
virt-install \
  --name talos-cp1 \
  --ram 4096 \
  --vcpus 4 \
  --disk /var/lib/libvirt/images/talos-cp1.qcow2,bus=virtio \
  --network bridge=br0,model=virtio \
  --os-variant generic \
  --boot hd \
  --noautoconsole \
  --import

# Create worker VM
virt-install \
  --name talos-worker1 \
  --ram 4096 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/talos-worker1.qcow2,bus=virtio \
  --network bridge=br0,model=virtio \
  --os-variant generic \
  --boot hd \
  --noautoconsole \
  --import
```

**Management commands:**

```bash
virsh list --all                    # List VMs
virsh start talos-cp1               # Start VM
virsh console talos-cp1             # Serial console
virsh domifaddr talos-cp1           # Get IP address
virsh shutdown talos-cp1            # Graceful shutdown
virsh destroy talos-cp1             # Force stop
virsh undefine talos-cp1 --remove-all-storage  # Delete VM
```

## Platform-Specific Gotchas

- **Disk path**: Use `/dev/vda` in machine config for virtio disks, not `/dev/sda`.
- **Bridge networking**: The default libvirt NAT network (`virbr0`) works for testing but does not support Talos VIP. Use a bridge for production.
- **Console access**: Use `virsh console` for serial access to the Talos maintenance mode output.
- **Nested virtualization**: If running libvirt inside a VM, enable nested virtualization on the host.
- **QEMU guest agent**: Include the extension for better integration. Socket setup: `--channel unix,target_type=virtio,name=org.qemu.guest_agent.0`.
