# Cilium CNI on Talos

Cilium is the recommended CNI for production Talos clusters. It provides an eBPF dataplane, optional kube-proxy replacement, L3-L7 NetworkPolicy, WireGuard encryption, and Gateway API support.

Reference: https://docs.siderolabs.com/kubernetes-guides/cni/deploying-cilium

## Machine Config Prerequisites

Set these in your machine configuration **before bootstrapping**:

### With kube-proxy (basic install)

```yaml
cluster:
  network:
    cni:
      name: none
```

### Without kube-proxy (kube-proxy replacement)

```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

Generate config with a patch:

```bash
talosctl gen config my-cluster https://mycluster.local:6443 --config-patch @patch.yaml
```

### KubePrism

KubePrism must be enabled. It exposes the API server locally on port 7445, which Cilium uses when replacing kube-proxy (`k8sServiceHost=localhost`, `k8sServicePort=7445`).

## Talos-Specific Requirements

These flags are **required for all Cilium installs on Talos**:

| Flag | Value | Reason |
|---|---|---|
| `securityContext.capabilities.ciliumAgent` | `{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}` | Required capabilities; `SYS_MODULE` is omitted — Talos does not allow workloads to load kernel modules |
| `securityContext.capabilities.cleanCiliumState` | `{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}` | Required for cleanup operations |
| `cgroup.autoMount.enabled` | `false` | Talos manages cgroup mounts |
| `cgroup.hostRoot` | `/sys/fs/cgroup` | Talos cgroup mount path |
| `ipam.mode` | `kubernetes` | Use Kubernetes IPAM |

## Installation Methods

### Method 1: Cilium CLI with kube-proxy

Cilium installs alongside the existing kube-proxy.

```bash
cilium install \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=false \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup
```

### Method 2: Cilium CLI without kube-proxy (kube-proxy replacement)

Requires `cluster.proxy.disabled: true` in machine config before bootstrap.

```bash
cilium install \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445
```

### Method 3: Cilium CLI with Gateway API

Extends Method 2 with Gateway API support. Required for HTTPRoute, GRPCRoute, and TLSRoute resources. Enable ALPN if using gRPC with GRPCRoutes over TLS.

```bash
cilium install \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445 \
    --set gatewayAPI.enabled=true \
    --set gatewayAPI.enableAlpn=true \
    --set gatewayAPI.enableAppProtocol=true
```

### Method 4: Helm direct install

Use Helm to install Cilium directly into the running cluster.

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

# With kube-proxy
helm install cilium cilium/cilium \
    --version 1.18.0 \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=false \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup

# Without kube-proxy — add these flags:
#   --set kubeProxyReplacement=true \
#   --set k8sServiceHost=localhost \
#   --set k8sServicePort=7445
```

### Method 5: Helm template + kubectl apply

Generate a static manifest from Helm, then apply it. Useful for GitOps workflows and auditing the full manifest before applying.

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm template cilium cilium/cilium \
    --version 1.18.0 \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=false \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup > cilium.yaml

kubectl apply -f cilium.yaml
```

Note: changing the namespace during `helm template` does not generate a manifest that creates the namespace itself — create it separately if needed.

### Method 6: Hosted manifest via cluster.network.cni.urls

Talos can fetch and apply a CNI manifest at bootstrap time via the machine config:

```yaml
cluster:
  network:
    cni:
      name: none
      urls:
        - https://example.com/cilium.yaml
```

**Security warning**: The Helm-generated Cilium manifest contains sensitive key material. Never host this manifest publicly. Prefer Method 7 (inline manifest) for bootstrap-time installation.

### Method 7: Inline manifest via cluster.inlineManifests (most secure)

Embed the Helm-templated manifest directly in the machine config. Talos applies it at bootstrap time. This keeps key material off the network.

```yaml
cluster:
  inlineManifests:
    - name: cilium
      contents: |
        # paste helm template output here
```

Important constraints:

- Add the inline manifest **only to control plane node configs** — all control plane nodes must have identical configuration.
- Talos only **creates** resources from inline manifests — it never deletes or updates them after initial apply.
- The namespace (`kube-system`) must already exist; `helm template` does not generate a namespace creation manifest when the namespace is changed.

## Known Issues

### DNS masquerading conflict

When `forwardKubeDNSToHost=true` is combined with `bpf.masquerade=true`, CoreDNS fails to resolve names.

Fix:

```bash
--set forwardKubeDNSToHost=false
```

### GCP internal load balancers

Local route configuration issues can occur with GCP internal LBs. Consult the Cilium GCP documentation for workarounds specific to your setup.

### Pod security violations in connectivity tests

`cilium connectivity test` may fail with pod security violations due to the `NET_RAW` capability requirement. Apply the privileged label to the test namespace:

```bash
kubectl label namespace cilium-test pod-security.kubernetes.io/enforce=privileged
```

### Bootstrap hang at phase 18/19

After setting `cni.name: none`, Talos will appear to hang at phase 18/19 during bootstrap. This is expected — Talos is waiting for a CNI to become ready. Install Cilium promptly after `talosctl bootstrap`. If CNI is not installed within ~10 minutes, Talos will reboot the node automatically.
