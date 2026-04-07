# Machine Config — Patching, Validation, and Extensions

This guide covers config patching strategies, the stock file workflow, secrets management, config validation, and system extensions via Image Factory. For config generation and structure, see the [main machine config guide](README.md).

---

## Config Patching

Patches let you customise configs without modifying the base files — essential for per-node differences (IPs, hostnames) or cluster-wide additions (CNI settings, extra kernel args).

### Strategic merge patches

The easiest format. Write a partial machine config YAML — only the fields you want to change. Talos merges it with the base config following these rules:

- Scalar values override the base value.
- List fields are **appended**, with important exceptions:
  - `cluster.network.podSubnets` and `cluster.network.serviceSubnets` — **overwritten**
  - `network.interfaces` — **merged by `interface:` or `deviceSelector:` key**
  - `network.interfaces[].vlans` — merged by `vlanId:`
  - `cluster.apiServer.auditPolicy` — **replaced entirely**
  - `ExtensionServiceConfig.configFiles` — merged by `mountPath`
- Use `$patch: delete` to remove a section.

```yaml
# patch-hostname.yaml — strategic merge, sets hostname and static IP
machine:
  network:
    hostname: worker-03
    interfaces:
      - interface: eth0
        addresses:
          - 10.0.0.23/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.0.1
```

```yaml
# patch-cilium.yaml — disable kube-proxy for Cilium replacement mode
cluster:
  proxy:
    disabled: true
  network:
    cni:
      name: none
```

```yaml
# patch-delete-example.yaml — remove a section entirely
machine:
  network:
    interfaces:
      - interface: eth1
        $patch: delete
```

### RFC 6902 JSON patches

Precise surgical edits using `op`/`path`/`value`. Can be written in YAML format. Use `replace` for existing paths and `add` for new paths.

```yaml
# patch-rfc6902.yaml
- op: replace
  path: /machine/network/hostname
  value: worker-03

- op: add
  path: /machine/network/interfaces/-
  value:
    interface: eth1
    dhcp: true

- op: replace
  path: /cluster/proxy/disabled
  value: true

- op: add
  path: /machine/sysctls/net.ipv4.ip_forward
  value: "1"
```

**Note:** JSON patches do not support multi-document machine configs.

### Application methods

```bash
# During generation — apply patches to all/CP/worker configs as they are created
talosctl gen config my-cluster https://10.0.0.100:6443 \
  --with-secrets secrets.yaml \
  --config-patch @all.yaml \
  --config-patch-control-plane @cp.yaml \
  --config-patch-worker @worker.yaml

# Post-generation (offline) — produce a new file without touching the base
talosctl machineconfig patch worker.yaml \
  --patch @patch-worker-03.yaml \
  -o worker-03.yaml

# On a running node (live apply, takes effect immediately or after reboot)
talosctl patch mc \
  --nodes 10.0.0.23 \
  --patch @patch.yaml
```

Multiple `--patch` flags are accepted; patches are applied sequentially left to right:

```bash
talosctl machineconfig patch worker.yaml \
  --patch @common.yaml \
  --patch @rack-b.yaml \
  --patch @worker-03-ip.yaml \
  -o worker-03.yaml
```

Inline patches (without `@`) are also accepted:

```bash
talosctl patch mc --nodes 10.0.0.11 \
  --patch '[{"op":"replace","path":"/machine/network/hostname","value":"cp-01"}]'
```

---

## The Stock File Philosophy

The standard workflow is:

1. Generate base configs — `controlplane.yaml` and `worker.yaml`.
2. Commit them immediately as **unmodified originals**:

   ```bash
   cp _out/controlplane.yaml controlplane.yaml.orig
   cp _out/worker.yaml worker.yaml.orig
   git add controlplane.yaml.orig worker.yaml.orig
   git commit -m "chore: commit stock Talos generated configs"
   ```

3. Apply patches to produce per-node configs — never edit the originals by hand.
4. Commit the patch files alongside the originals.

This makes customisations explicit and reviewable:

```bash
# See exactly what you changed from the stock config
diff <(talosctl machineconfig patch controlplane.yaml.orig --patch /dev/null) \
     <(talosctl machineconfig patch controlplane.yaml.orig --patch @cp.yaml)

# Or via git after applying patches over the originals
git diff controlplane.yaml.orig controlplane.yaml
```

Benefits:
- Upgrading Talos is: `talosctl gen config` (new version) → re-apply patches → review diff → apply.
- Onboarding is: read the originals (understand defaults) → read the patches (understand customisations).
- Debugging is: isolate whether an issue is in the base config or a patch.

---

## Secrets Management

`secrets.yaml` is the root of trust for a Talos cluster. It contains:

| Material | Purpose |
|----------|---------|
| Machine CA cert + key | Signs node TLS certificates |
| Etcd CA cert + key | Signs etcd peer and client certs |
| Kubernetes CA cert + key | Signs all in-cluster TLS (kubelet, API server) |
| Service account key | Signs Kubernetes service account tokens |
| Bootstrap token | Used to join nodes to the cluster |
| Secretbox encryption key | Encrypts etcd secrets at rest (if enabled) |

**Storage:** treat `secrets.yaml` like a private key — never commit it to a public repository. Options:

- Encrypted file in git (SOPS + age or GPG)
- Secret manager (HashiCorp Vault, AWS Secrets Manager, 1Password)
- Encrypted local volume or hardware token

**Regenerating configs without losing cluster identity:**

As long as you have `secrets.yaml`, you can regenerate `controlplane.yaml` and `worker.yaml` at any time — for upgrades, adding SANs, or changing settings — and the new configs will be cryptographically consistent with the running cluster.

```bash
# Safe upgrade workflow — secrets preserve cluster identity
talosctl gen config homelab https://10.0.0.100:6443 \
  --with-secrets secrets.yaml \
  --kubernetes-version 1.32.0 \
  --output-dir _out \
  --config-patch @all.yaml \
  --config-patch-control-plane @cp.yaml \
  --config-patch-worker @worker.yaml
```

**Bootstrap token rotation:** the token in `secrets.yaml` is only used during initial bootstrap. Once the cluster is up, the token is no longer needed for day-to-day operation. Secrets can be rotated post-bootstrap via `talosctl rotate-ca`.

---

## Config Validation

Validate configs before applying them to catch schema errors, missing required fields, and mode-specific issues.

```bash
# Validate a control plane config for bare-metal
talosctl validate --config _out/controlplane.yaml --mode metal

# Validate a worker config for a cloud VM
talosctl validate --config _out/worker.yaml --mode cloud

# Validate for a container runtime (talosctl cluster create)
talosctl validate --config controlplane.yaml --mode container
```

**Validation modes:**

| Mode | When to use |
|------|-------------|
| `metal` | Physical servers, VMs with full disk access (most common) |
| `cloud` | Cloud instances where disk/network may differ (AWS, GCP, Azure, Hetzner) |
| `container` | Local `talosctl cluster create` containers |

Validation checks the config against the `v1alpha1` schema, verifies required fields are present for the given mode, and catches common errors like invalid CIDR notation or missing cluster endpoint. It does not contact a live node.

```bash
# Validate all generated configs at once
for f in _out/*.yaml; do
  echo "--- $f ---"
  talosctl validate --config "$f" --mode metal
done
```

---

## System Extensions

Talos Linux is immutable — you cannot install packages at runtime. System extensions are the mechanism for adding software (kernel modules, userspace tools, firmware) to the OS image at build time.

### Image Factory

The official Image Factory at `https://factory.talos.dev` builds custom Talos installer and ISO images from a **schematic** — a YAML document describing what extensions to include.

**Schematic format:**

```yaml
customization:
  extraKernelArgs:
    - vga=791
    - net.ifnames=0
  systemExtensions:
    officialExtensions:
      - siderolabs/iscsi-tools
      - siderolabs/qemu-guest-agent
      - siderolabs/intel-ucode
```

Submit the schematic to the factory API to get a content-addressable schematic ID:

```bash
curl -X POST \
  --data-binary @schematic.yaml \
  https://factory.talos.dev/schematics
# Returns: {"id":"abc123..."}
```

Use the schematic ID to construct image URLs:

```bash
# Installer image for machine.install.image
factory.talos.dev/installer/<schematic-id>:v1.9.0

# ISO download
https://factory.talos.dev/image/<schematic-id>/v1.9.0/metal-amd64.iso

# iPXE / PXE boot assets
https://factory.talos.dev/image/<schematic-id>/v1.9.0/kernel-amd64
https://factory.talos.dev/image/<schematic-id>/v1.9.0/initramfs-amd64.xz
```

The "vanilla" (no extensions) schematic ID is always:
`376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba`

### Common extensions

| Extension | Use case |
|-----------|---------|
| `siderolabs/iscsi-tools` | Required for iSCSI-backed storage (Longhorn, OpenEBS iSCSI) |
| `siderolabs/qemu-guest-agent` | QEMU/KVM guest agent for VM management (Proxmox, libvirt) |
| `siderolabs/intel-ucode` | Intel CPU microcode updates — apply on Intel hosts |
| `siderolabs/amd-ucode` | AMD CPU microcode updates — apply on AMD hosts |
| `siderolabs/zfs` | ZFS kernel modules for ZFS-backed storage |
| `siderolabs/nvidia-container-toolkit` | NVIDIA GPU support |
| `siderolabs/gvisor` | gVisor container runtime for isolation |
| `siderolabs/kata-containers` | Kata Containers runtime |
| `siderolabs/drbd` | DRBD kernel module for LINSTOR/Piraeus storage |
| `siderolabs/nfs-common` | NFS client support |

### Using the extension in machine config

Reference the factory installer image in `machine.install.image`:

```yaml
machine:
  install:
    image: factory.talos.dev/installer/abc123schematicid:v1.9.0
```

For upgrades, use the same schematic ID with the new version:

```bash
talosctl upgrade \
  --nodes 10.0.0.11 \
  --image factory.talos.dev/installer/<schematic-id>:v1.9.2
```

### Checking installed extensions

```bash
talosctl get extensions --nodes 10.0.0.11
```

### Extension notes

- Extensions are baked into the installer image — changing extensions requires re-installing or upgrading via a new factory image.
- `siderolabs/iscsi-tools` is needed at the OS level even when the CSI driver runs in a container — the kernel modules must be present.
- Kernel args and META values in a schematic are **ignored** for installer images; only `systemExtensions` takes effect in installer/initramfs images.
- Each Talos version has its own extension compatibility matrix — check the factory UI for supported extensions per version.
