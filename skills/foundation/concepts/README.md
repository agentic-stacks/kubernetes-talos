# Talos Linux Architecture: Core Concepts

Agent reference for understanding Talos Linux internals. Use this before executing any node configuration, upgrade, or troubleshooting task.

---

## What Is Talos Linux

Talos Linux is an immutable, API-managed operating system built exclusively to run Kubernetes. It eliminates the entire traditional Linux administration surface:

- **No SSH** — remote shell access does not exist
- **No interactive shell** — there is no bash, sh, or login prompt
- **No package manager** — apt, yum, dnf, and equivalents are absent
- **No init scripts** — no systemd units for user-defined services
- **No direct file editing** — the root filesystem is read-only squashfs

All interaction with a Talos node happens through the gRPC API via the `talosctl` CLI. Every configuration change is declarative: you apply a machine config and Talos converges to the desired state.

**Where Talos runs:**
- Bare metal (x86_64, arm64)
- Virtual machines (VMware, Hyper-V, KVM/QEMU, Proxmox)
- Cloud platforms (AWS, GCP, Azure, Hetzner, Equinix Metal, Oracle Cloud)
- Edge and single-board computers (Raspberry Pi, Jetson)

**What Talos does not do:** run arbitrary workloads outside of Kubernetes. The OS exists solely to host a Kubernetes node.

---

## Architecture Overview

Talos replaces the traditional Linux userspace with a minimal set of purpose-built daemons. Every service is managed by machined, not systemd.

### Core System Services

| Service | Role | Port |
|---------|------|------|
| `machined` | PID 1 — machine lifecycle, config convergence, service supervisor | internal |
| `apid` | gRPC API gateway — authenticates and proxies all talosctl requests | TCP 50000 |
| `trustd` | Certificate authority and trust delegation for cluster bootstrap PKI | TCP 50001 |
| `containerd` | Container runtime for both Talos system services and Kubernetes pods | internal |
| `kubelet` | Kubernetes node agent — managed as a Talos service | internal |
| `etcd` | Distributed key-value store for Kubernetes cluster state | TCP 2379/2380 |
| `networkd` | Network configuration daemon | internal |
| `udevd` | Device event management | internal |

### machined (PID 1)

`machined` is the Talos init process. It is not systemd. Responsibilities:

- Applies machine configuration on boot and on `talosctl apply-config`
- Manages the lifecycle of all system services (start, stop, restart)
- Implements the controller/resource model that drives state convergence
- Exposes an internal API that `apid` proxies to the outside world

machined rejects arbitrary user-defined services by design. You cannot add a custom daemon to Talos.

### apid (port 50000)

`apid` is the sole external API entry point. Every `talosctl` command reaches the node through apid. Its role:

- Terminates mutual TLS (mTLS) client connections from talosctl
- Routes requests to `machined` for local operations
- Proxies requests to other nodes' `apid` instances for cluster-wide commands (e.g., `talosctl --nodes` targeting multiple nodes)

You never talk to machined directly from outside the node.

### trustd (port 50001)

`trustd` implements the Root of Trust for cluster bootstrap. It enables:

- PKI data distribution during control plane initialisation
- Accepting write requests from peer nodes to place certificate material on disk
- Bootstrapping the cluster CA chain without external secret injection

trustd runs on control plane nodes. It is active during bootstrap and certificate rotation.

### etcd

Runs on control plane nodes only. Talos manages etcd as a system service — it does not require manual etcd installation or configuration. Members are added and removed automatically during node join and removal.

### kubelet

Managed as a Talos service. Talos controls the kubelet binary version, configuration, and lifecycle. You do not interact with kubelet directly — all kubelet configuration passes through the machine config `machine.kubelet` stanza.

### containerd

Two namespaces:

| Namespace | Purpose |
|-----------|---------|
| `system` | Talos system services (e.g., kubelet, etcd, trustd) |
| `k8s.io` | Kubernetes pod workloads |

This separation means Talos system containers are isolated from Kubernetes workload containers.

---

## The Machine Config Model

Talos is entirely declarative. The machine config is a YAML document conforming to the `v1alpha1` schema.

### Two Config Types

| Type | Used For |
|------|----------|
| `controlplane.yaml` | Nodes that run etcd, kube-apiserver, kube-controller-manager, kube-scheduler |
| `worker.yaml` | Nodes that run only kubelet and container runtime |

### Config Generation

```
talosctl gen config <cluster-name> https://<control-plane-endpoint>:6443
```

This produces `controlplane.yaml`, `worker.yaml`, and `talosconfig` (client credentials).

### Config Structure

```yaml
version: v1alpha1
machine:
  type: controlplane   # or worker
  token: <join-token>
  ca:
    crt: <base64>
    key: <base64>
  network: { ... }
  install: { ... }
  kubelet: { ... }
  encrypt: { ... }     # optional LUKS2 disk encryption
cluster:
  controlPlane:
    endpoint: https://...
  clusterName: ...
  network: { ... }
  etcd: { ... }
```

### Declarative Convergence

Talos does not accept imperative commands like "install package X" or "edit /etc/foo". You modify the machine config and apply it. Talos computes the diff and converges:

- **No in-place mutation** of OS files
- **No shell scripts** run on the node
- **No Ansible playbooks** touching the OS layer
- Config is stored on the STATE partition and re-applied on every boot

---

## Security Model

### Mutual TLS Everywhere

All communication between talosctl and a Talos node, and between nodes, uses mTLS. Both sides present certificates; connections without valid client certs are rejected.

### talosconfig

The client credential file (`~/.talos/config` by default) contains:

- CA certificate (to verify the node's server cert)
- Client certificate (presented to authenticate the operator)
- Client private key

Without a valid talosconfig, no API calls can succeed. There is no username/password fallback.

### No Privilege Escalation

- No root login
- No sudo
- No su
- No SSH keys to exfiltrate
- No interactive access of any kind

### Disk Encryption

Optional LUKS2 full-disk encryption for the STATE and EPHEMERAL partitions. Key management options:

| Method | Description |
|--------|-------------|
| `nodeID` | Key derived from unique node hardware identifiers |
| `KMS` | Key sealed by an external Key Management Server |
| `TPM` | Key sealed by the node's Trusted Platform Module |
| `static` | Passphrase-based (not recommended for production) |

### Secure Boot

Talos supports UEFI Secure Boot via signed unified kernel images (UKI). The entire boot chain — firmware → bootloader → kernel → initramfs — is cryptographically verified.

### Kubernetes Secrets Encryption at Rest

Talos configures Kubernetes with Secretbox encryption for etcd secret data. Secrets stored in etcd are encrypted with a symmetric key managed in the machine config.

### Maintenance Mode

During initial boot before a machine config is applied, the node runs in **maintenance mode**. In this state:

- The API is accessible on port 50000 with server-only TLS (no client cert required)
- Only `talosctl apply-config` and `talosctl get` are permitted
- No Kubernetes services run

Once a config is applied, the node exits maintenance mode and mTLS is enforced for all subsequent requests.

---

## Filesystem and Partitions

Talos uses a fixed partition layout. Partitions are created at install time and cannot be resized without reinstalling.

### Partition Table

| Partition | Size | Mount Point | Purpose |
|-----------|------|-------------|---------|
| EFI System Partition | 100 MB | `/boot/EFI` | UEFI boot data, bootloader (GRUB or systemd-boot) |
| BIOS Boot | 1 MB | none | GRUB second-stage for legacy BIOS systems |
| BOOT | 1 GB | `/boot` | Kernel and initramfs, A/B slot scheme for atomic upgrades and rollback |
| META | 1 MB | none | Key-value metadata store for node identifiers, upgrade state, and reset flags |
| STATE | 100 MB | `/system/state` | Machine config (optionally LUKS2 encrypted), node identity, KubeSpan data |
| EPHEMERAL | remainder | `/var` | Kubernetes workload data, container images, etcd data, logs |

### A/B Boot Scheme

The BOOT partition holds two slots (A and B). On upgrade:

1. New kernel and initramfs are written to the inactive slot
2. The bootloader is updated to boot from the new slot
3. On rollback, the bootloader switches back to the previous slot

This makes OS upgrades atomic and recoverable without reinstalling.

### Filesystem Layers

| Layer | Type | Writable |
|-------|------|----------|
| `/` (root) | squashfs (read-only) | No |
| `/system` | tmpfs overlay | Yes (in-memory, lost on reboot) |
| `/var` | EPHEMERAL partition (ext4) | Yes (persistent) |
| `/etc` | overlayfs over squashfs | Selected files persisted to STATE |

All files under `/system` are recreated on each boot from the squashfs image. You cannot persist changes to `/system` by writing files — you must apply a machine config.

---

## Lifecycle Concepts

### Node States

Talos nodes move through well-defined states:

| State | Description |
|-------|-------------|
| `maintenance` | Node booted, no machine config applied. API accessible without client cert. Awaits `apply-config`. |
| `installing` | Machine config received. Talos is writing itself to disk, partitioning, and configuring. |
| `running` | Node is fully operational. All services started. Kubernetes node is Ready. |
| `rebooting` | Transitional — node executing an ordered reboot. |
| `shutting down` | Transitional — node executing an ordered shutdown. |
| `resetting` | Node is wiping disk state, removing machine config, returning toward factory state. |
| `upgrading` | Node is applying a Talos OS upgrade (writes new image to inactive boot slot). |

### Config Apply Modes

When running `talosctl apply-config`, the `--mode` flag controls how the change is applied:

| Mode | Flag | Behaviour | When to Use |
|------|------|-----------|-------------|
| auto | `--mode=auto` | Talos determines whether a reboot is required | Default; safe for most changes |
| no-reboot | `--mode=no-reboot` | Applies change in place; **fails** if the change requires a reboot | Config changes known to be live-applicable |
| reboot | `--mode=reboot` | Applies change and immediately reboots the node | Changes known to require reboot |
| staged | `--mode=staged` | Writes change to disk but does not apply until next reboot | Maintenance windows, batch changes |
| try | `--mode=try` | Applies change temporarily; **auto-rolls back** after timeout (default 1 minute) if not confirmed | Network config changes where misconfiguration would lose access |

### Changes That Require a Reboot

| Change Type | Reboot Required |
|-------------|----------------|
| Kernel parameters (`machine.install.extraKernelArgs`) | Yes |
| Disk encryption enable/disable | Yes |
| Install disk change | Yes |
| Hostname | No (live-applied) |
| Network interface configuration | Sometimes (depends on driver) |
| Kubelet extra args | No |
| Extra manifests | No |
| Trusted CA certificates | No |
| Talos OS version upgrade | Yes |
| System extensions | Yes |
| Time servers (NTP) | No |

When in doubt, use `--mode=auto`. Talos will reboot only if required.

### Upgrade Lifecycle

```
talosctl upgrade --nodes <node-ip> --image ghcr.io/siderolabs/installer:<version>
```

1. New installer image pulled
2. New kernel + initramfs written to inactive BOOT slot
3. Node reboots into new slot
4. If boot succeeds, new slot becomes active
5. If boot fails, bootloader reverts to previous slot automatically

Control plane upgrades should be performed one node at a time to maintain etcd quorum.

---

## Key Differences from Traditional Kubernetes Deployments

| Capability | Talos Linux | kubeadm | k3s | RKE2 |
|-----------|-------------|---------|-----|------|
| **SSH access** | None — no SSH daemon | Required for setup | Required for setup | Required for setup |
| **Interactive shell** | None | bash/sh available | bash/sh available | bash/sh available |
| **Package management** | None (immutable OS) | apt/yum on host | apt/yum on host | apt/yum on host |
| **Config method** | Declarative YAML (v1alpha1), applied via gRPC API | Imperative kubeadm CLI + manual host config | Installer flags + systemd unit | Installer flags + systemd unit |
| **Init system** | machined (custom, Kubernetes-only) | systemd | systemd (openrc on Alpine) | systemd |
| **OS upgrade method** | Atomic A/B image swap via `talosctl upgrade` | Manual OS package updates | Manual OS package updates | Manual OS package updates |
| **Kubernetes upgrade** | `talosctl upgrade-k8s` — coordinated, rolling | Manual kubeadm upgrade steps | In-place binary replacement | In-place binary replacement |
| **Security model** | mTLS for all access, no root login, optional LUKS2 + Secure Boot | SSH + sudo, standard Linux permissions | SSH + sudo, optional TLS | SSH + sudo, enforces PSP/PSA |
| **etcd management** | Managed automatically by Talos | Managed by kubeadm, manual for disaster recovery | Uses embedded SQLite/etcd (k3s default: SQLite) | Uses embedded etcd, managed by RKE2 |
| **CNI** | Flannel (default), any CNI via machine config | User-supplied | Flannel (built-in) | Canal (built-in) |
| **Supported platforms** | Bare metal, VMs, cloud, edge | Any Linux host | Any Linux host | Any Linux host |
| **API for node management** | gRPC (talosctl) | SSH + kubectl | SSH + kubectl | SSH + kubectl |
| **Config drift prevention** | Enforced — immutable OS, convergence on every boot | Not enforced — host is mutable | Not enforced — host is mutable | Not enforced — host is mutable |

### The Fundamental Distinction

kubeadm, k3s, and RKE2 install Kubernetes **onto** a general-purpose Linux host. The host remains fully mutable — you can SSH in, install packages, edit files, and break things in ways that are invisible to Kubernetes.

Talos **is** the node. The OS and the Kubernetes node are a single, atomically managed unit. There is no "host OS" layer that exists independently of the cluster.

---

## Quick Reference: talosctl Commands for Agents

| Task | Command |
|------|---------|
| Apply config (auto mode) | `talosctl apply-config --nodes <ip> --file controlplane.yaml` |
| Apply with staged mode | `talosctl apply-config --nodes <ip> --file controlplane.yaml --mode=staged` |
| Check node state | `talosctl get machinestatus --nodes <ip>` |
| View running services | `talosctl services --nodes <ip>` |
| Read service logs | `talosctl logs <service> --nodes <ip>` |
| Upgrade Talos OS | `talosctl upgrade --nodes <ip> --image ghcr.io/siderolabs/installer:<ver>` |
| Upgrade Kubernetes | `talosctl upgrade-k8s --to <k8s-version>` |
| Reset node | `talosctl reset --nodes <ip> --graceful=false --reboot` |
| Bootstrap etcd | `talosctl bootstrap --nodes <first-control-plane-ip>` |
| Generate config | `talosctl gen config <name> https://<endpoint>:6443` |
| Retrieve kubeconfig | `talosctl kubeconfig --nodes <ip>` |

**Source:** https://docs.siderolabs.com/talos/v1.9/overview/what-is-talos
