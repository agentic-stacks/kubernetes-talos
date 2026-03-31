# Machine Config — Talos Config Generation and Management

This is the most critical skill in the stack. Every node's behaviour is expressed entirely in its machine config YAML — there is no SSH, no shell, no ad-hoc changes. Getting config generation and patching right is the foundation everything else builds on.

---

## Generating Configs

Config generation requires two steps: generate a secrets bundle first, then generate the machine configs from it.

```bash
# Step 1 — generate secrets (root CAs, bootstrap tokens, encryption keys)
talosctl gen secrets -o secrets.yaml

# Step 2 — generate machine configs
talosctl gen config <cluster-name> <cluster-endpoint> \
  --with-secrets secrets.yaml \
  --output-dir _out \
  --kubernetes-version 1.31.0 \
  --install-disk /dev/sda
```

**Example:**

```bash
talosctl gen config homelab https://10.0.0.100:6443 \
  --with-secrets secrets.yaml \
  --output-dir _out \
  --kubernetes-version 1.31.0 \
  --install-disk /dev/nvme0n1
```

**Output files:**

| File | Purpose |
|------|---------|
| `controlplane.yaml` | Machine config applied to every control plane node |
| `worker.yaml` | Machine config applied to every worker node |
| `talosconfig` | Client credentials for `talosctl` — analogous to `kubeconfig` |

**Why `--with-secrets` matters:**

Without `--with-secrets`, each `talosctl gen config` run generates a brand-new PKI hierarchy and bootstrap token set. Re-generating without the same secrets file produces configs that are cryptographically incompatible with an already-bootstrapped cluster — the new certs won't be signed by the same CA the cluster trusts. Always generate secrets once, store them securely, and pass them on every subsequent config regeneration.

**Other useful generation flags:**

```bash
# Specify Talos installer image (e.g. from Image Factory with extensions)
--talos-version v1.9.0
--install-image factory.talos.dev/installer/<schematic-id>:v1.9.0

# Apply patches during generation (see Config Patching section)
--config-patch @all.yaml
--config-patch-control-plane @cp.yaml
--config-patch-worker @worker.yaml

# Kubernetes API server additional SANs
# (also set via machine.certSANs in config)
```

---

## Config Structure Deep Dive

Talos machine configs are `v1alpha1` YAML documents. A single file covers all settings for a node — OS install, networking, kubelet behaviour, and cluster participation.

### Top-level shape

```yaml
version: v1alpha1
debug: false
persist: true

machine:
  type: controlplane        # or: worker
  token: <bootstrap-token>
  ca:
    crt: <base64-encoded-CA-cert>
    key: <base64-encoded-CA-key>
  # ... all machine-level settings

cluster:
  id: <cluster-id>
  secret: <cluster-secret>
  controlPlane:
    endpoint: https://10.0.0.100:6443
  # ... all cluster-level settings
```

Worker configs omit `machine.ca.key` and most cluster secrets.

---

### machine.type

```yaml
machine:
  type: controlplane   # runs etcd + kube-apiserver + kube-controller-manager + kube-scheduler
  # or
  type: worker         # runs kubelet only
```

---

### machine.network

Controls everything about OS-level networking.

```yaml
machine:
  network:
    hostname: cp-01
    nameservers:
      - 1.1.1.1
      - 8.8.8.8
    searchDomains:
      - cluster.local
      - home.internal
    extraHostEntries:
      - ip: 10.0.0.50
        aliases:
          - registry.internal

    interfaces:
      # Static IP on a named interface
      - interface: eth0
        addresses:
          - 10.0.0.11/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.0.1
            metric: 1024
        mtu: 1500

      # DHCP on a second interface
      - interface: eth1
        dhcp: true
        dhcpOptions:
          routeMetric: 2048

      # Virtual IP — floats to the current etcd leader
      - interface: eth0
        vip:
          ip: 10.0.0.100

      # Bond
      - interface: bond0
        bond:
          mode: 802.3ad
          lacpRate: fast
          miimon: 100
          updelay: 200
          downdelay: 200
        interfaces:
          - eth0
          - eth1
        addresses:
          - 10.0.0.11/24

      # VLAN on top of an interface
      - interface: eth0
        vlans:
          - vlanId: 100
            addresses:
              - 10.100.0.11/24
            dhcp: false

      # Device selector (match by driver, useful when interface names vary)
      - deviceSelector:
          driver: virtio_net
          physical: true
        addresses:
          - 10.0.0.11/24
```

---

### machine.install

Controls what gets written to disk at install time.

```yaml
machine:
  install:
    disk: /dev/sda

    # Or use a selector when disk name is unpredictable
    diskSelector:
      size: ">= 100GB"
      type: nvme        # ssd | hdd | nvme | sd

    image: ghcr.io/siderolabs/installer:v1.9.0
    # For Image Factory installer with extensions:
    # image: factory.talos.dev/installer/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba:v1.9.0

    wipe: false

    extraKernelArgs:
      - console=tty0
      - console=ttyS0,115200n8
      - net.ifnames=0

    # Extensions via install image (preferred — use Image Factory schematic instead)
    extensions:
      - image: ghcr.io/siderolabs/iscsi-tools:v0.1.4
```

---

### machine.certSANs

Additional Subject Alternative Names added to the node's certificate. Critical for API server access via load balancers, VIPs, or DNS names that aren't auto-detected.

```yaml
machine:
  certSANs:
    - 10.0.0.100          # VIP or LB IP
    - api.homelab.internal # DNS name for the cluster endpoint
    - 10.0.0.11            # this node's own IP (often auto-included)
```

These SANs are baked into the TLS certificate at bootstrap. If you need to add a new SAN later, you must regenerate configs and rotate certificates.

---

### machine.kubelet

```yaml
machine:
  kubelet:
    image: ghcr.io/siderolabs/kubelet:v1.31.0

    extraArgs:
      rotate-server-certificates: true
      node-labels: "topology.kubernetes.io/zone=rack-a"

    # Restrict kubelet's node IP to a specific subnet
    # Useful when nodes have multiple interfaces
    nodeIP:
      validSubnets:
        - 10.0.0.0/24
        - "!192.168.0.0/16"   # exclude prefix with !

    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - bind
          - rshared
          - rw

    extraConfig:
      maxPods: 150
      featureGates:
        GracefulNodeShutdown: true
```

---

### machine.features

```yaml
machine:
  features:
    rbac: true
    stableHostname: true       # hostname derived from node identity, not random

    # Allow pods in specific namespaces to call the Talos API
    kubernetesTalosAPIAccess:
      enabled: true
      allowedRoles:
        - os:reader
      allowedKubernetesNamespaces:
        - kube-system

    apidCheckExtKeyUsage: true
    diskQuotaSupport: true

    # KubePrism: local proxy for kube-apiserver on every node
    kubePrism:
      enabled: true
      port: 7445

    # Host-level DNS cache resolver
    hostDNS:
      enabled: true
      forwardKubeDNSToHost: false
      resolveMemberNames: true
```

---

### machine.sysctls

Kernel parameter overrides applied at boot.

```yaml
machine:
  sysctls:
    net.ipv4.ip_forward: "1"
    net.bridge.bridge-nf-call-iptables: "1"
    net.bridge.bridge-nf-call-ip6tables: "1"
    kernel.panic: "10"
    vm.nr_hugepages: "1024"
```

---

### machine.registries

```yaml
machine:
  registries:
    # Mirror docker.io through a local registry
    mirrors:
      docker.io:
        endpoints:
          - https://registry.internal/v2/docker-proxy
        overridePath: true
      registry.k8s.io:
        endpoints:
          - https://registry.internal/v2/k8s-proxy

    # Authenticate against a private registry
    config:
      registry.internal:
        auth:
          username: robot$talos
          password: supersecret
        tls:
          insecureSkipVerify: false
          ca:
            crt: <base64-encoded-CA>
```

---

### machine.systemDiskEncryption

Encrypts the `STATE` and `EPHEMERAL` partitions.

```yaml
machine:
  systemDiskEncryption:
    state:
      provider: luks2
      keys:
        - nodeID: {}   # key derived from node UUID — survives reboots, lost if hardware changes
          slot: 0
      cipher: aes-xts-plain64
      keySize: 256

    ephemeral:
      provider: luks2
      keys:
        - tpm: {}      # key sealed in TPM — requires secure boot for full benefit
          slot: 0
```

---

### cluster.controlPlane.endpoint

The shared endpoint all nodes and clients use to reach the API server. This is baked into certificates at bootstrap — changing it later requires cert rotation.

```yaml
cluster:
  controlPlane:
    endpoint: https://10.0.0.100:6443   # VIP or external LB
    localAPIServerPort: 6443            # port kube-apiserver binds on each node
```

Use a VIP (configured via `machine.network.interfaces[].vip`), external load balancer, or DNS name. Single-node clusters can use the node IP directly.

---

### cluster.network

```yaml
cluster:
  network:
    dnsDomain: cluster.local

    podSubnets:
      - 10.244.0.0/16      # overwritten (not appended) by patches

    serviceSubnets:
      - 10.96.0.0/12       # overwritten (not appended) by patches

    cni:
      name: none           # disable built-in Flannel when using Cilium or Calico
      # name: flannel      # use built-in Flannel
      # name: custom
      # urls:
      #   - https://raw.githubusercontent.com/cilium/cilium/main/install/kubernetes/quick-install.yaml
```

---

### cluster.etcd

```yaml
cluster:
  etcd:
    # Restrict etcd peer/client traffic to a specific subnet
    advertisedSubnets:
      - 10.0.0.0/24

    extraArgs:
      election-timeout: "2000"
      heartbeat-interval: "500"
```

---

### cluster.proxy

```yaml
cluster:
  proxy:
    disabled: true    # set true when using Cilium in kube-proxy replacement mode
    # image: registry.k8s.io/kube-proxy:v1.31.0
    # mode: iptables  # or ipvs
    # extraArgs:
    #   metrics-bind-address: 0.0.0.0:10249
```

---

### cluster.apiServer

```yaml
cluster:
  apiServer:
    image: registry.k8s.io/kube-apiserver:v1.31.0

    certSANs:
      - 10.0.0.100
      - api.homelab.internal

    extraArgs:
      audit-log-path: /var/log/audit/kube-apiserver.log
      audit-log-maxage: "30"
      oidc-issuer-url: https://dex.homelab.internal
      oidc-client-id: kubernetes

    admissionControl:
      - name: PodSecurity
        configuration:
          apiVersion: pod-security.admission.config.k8s.io/v1alpha1
          kind: PodSecurityConfiguration
          defaults:
            enforce: baseline
            audit: restricted
            warn: restricted

    auditPolicy:
      apiVersion: audit.k8s.io/v1
      kind: Policy
      rules:
        - level: Metadata
```

---

### cluster.inlineManifests and cluster.extraManifests

Applied once at cluster bootstrap. Useful for deploying CNI, storage operators, or namespaces before any external tooling runs.

```yaml
cluster:
  # Inline YAML embedded in the config
  inlineManifests:
    - name: cilium-namespace
      contents: |
        apiVersion: v1
        kind: Namespace
        metadata:
          name: cilium
          labels:
            pod-security.kubernetes.io/enforce: privileged

  # URLs fetched and applied at bootstrap
  extraManifests:
    - https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml
```

---

### cluster.discovery

```yaml
cluster:
  discovery:
    enabled: true
    registries:
      kubernetes:
        disabled: false    # uses Kubernetes API for discovery between CP nodes
      service:
        disabled: false    # uses the Sidero discovery service (public, encrypted)
        endpoint: https://discovery.talos.dev
```

---

### cluster.allowSchedulingOnControlPlanes

```yaml
cluster:
  allowSchedulingOnControlPlanes: true   # required for single-node or 3-node all-in-one clusters
```

---

### cluster.secretboxEncryptionSecret

Enables encryption-at-rest for Kubernetes secrets stored in etcd using the secretbox cipher.

```yaml
cluster:
  secretboxEncryptionSecret: <base64-encoded-32-byte-key>
  # Generate with: openssl rand -base64 32
```

---

### Complete annotated example

```yaml
version: v1alpha1
debug: false
persist: true

machine:
  type: controlplane
  token: abc123.bootstraptoken
  ca:
    crt: <base64>
    key: <base64>

  certSANs:
    - 10.0.0.100
    - api.homelab.internal

  kubelet:
    extraArgs:
      rotate-server-certificates: true
    nodeIP:
      validSubnets:
        - 10.0.0.0/24

  network:
    hostname: cp-01
    nameservers:
      - 1.1.1.1
    interfaces:
      - interface: eth0
        addresses:
          - 10.0.0.11/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.0.1
        vip:
          ip: 10.0.0.100

  install:
    disk: /dev/sda
    image: factory.talos.dev/installer/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba:v1.9.0
    wipe: false

  features:
    rbac: true
    stableHostname: true
    kubePrism:
      enabled: true
      port: 7445

  sysctls:
    net.ipv4.ip_forward: "1"

cluster:
  id: <cluster-id>
  secret: <cluster-secret>

  controlPlane:
    endpoint: https://10.0.0.100:6443

  clusterName: homelab

  network:
    dnsDomain: cluster.local
    podSubnets:
      - 10.244.0.0/16
    serviceSubnets:
      - 10.96.0.0/12
    cni:
      name: none

  proxy:
    disabled: true

  etcd:
    ca:
      crt: <base64>
      key: <base64>
    advertisedSubnets:
      - 10.0.0.0/24

  apiServer:
    certSANs:
      - 10.0.0.100
    extraArgs:
      audit-log-path: /var/log/audit/kube-apiserver.log

  discovery:
    enabled: true
    registries:
      kubernetes:
        disabled: false
      service:
        disabled: false

  allowSchedulingOnControlPlanes: false

  secretboxEncryptionSecret: <base64-32-byte-key>
```

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
