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

## Next Steps

- [Config patching, validation, and extensions](patching-and-extensions.md) — strategic merge patches, RFC 6902, stock file workflow, secrets management, config validation, and system extensions via Image Factory
