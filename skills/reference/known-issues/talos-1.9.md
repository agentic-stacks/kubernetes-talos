# Known Issues — Talos 1.9.x

---

### Network Interface Name Changes (systemd-udev Replacing eudev)

**Symptom:** After upgrading to Talos 1.9.x, network interfaces are renamed. Interfaces previously named `eth0` may become `enp0s3`, `ens3`, or similar predictable names. Machine config referencing `eth0` by name stops working, and the node loses network connectivity after upgrade.

**Cause:** Talos 1.9 replaced `eudev` with `systemd-udevd`. The new udev implementation uses predictable interface naming by default, which assigns names based on hardware topology (bus path, slot, MAC address) rather than kernel enumeration order.

**Workaround:** Use `deviceSelector` instead of hard-coding interface names in machine config. `deviceSelector` matches interfaces by hardware properties that survive renames:

```yaml
machine:
  network:
    interfaces:
      # WRONG - fragile, breaks on udev changes:
      # - interface: eth0
      #   addresses:
      #     - 10.0.0.10/24

      # CORRECT - survives interface renames:
      - deviceSelector:
            busPath: "0000:01:00.0"    # PCI bus path (most stable)
          addresses:
            - 10.0.0.10/24

      # Alternative selectors:
      - deviceSelector:
            hardwareAddr: "aa:bb:cc:dd:ee:ff"   # MAC address
          addresses:
            - 10.0.0.11/24

      - deviceSelector:
            driver: virtio_net    # Kernel driver name
          addresses:
            - 10.0.0.12/24
```

To find the correct selector values before upgrading:

```bash
# List all links with bus paths and hardware addresses
talosctl get links -n <node> -o yaml
```

**Affected versions:** 1.9.0+

**Status:** By design. `deviceSelector` is the recommended approach going forward for all Talos versions. Hard-coded interface names should be considered deprecated practice.

---

### Cilium DNS Masquerading Breaks with forwardKubeDNSToHost + bpf.masquerade

**Symptom:** DNS resolution fails for pods when Cilium is installed with both `forwardKubeDNSToHost: true` (or left as default) and `bpf.masquerade: true`. Pods cannot resolve cluster-internal service names. `cilium connectivity test` shows DNS-related failures.

**Cause:** When `bpf.masquerade` is enabled, Cilium handles masquerading in eBPF. Combined with `forwardKubeDNSToHost: true`, DNS traffic is forwarded to the host network namespace for resolution, but the BPF masquerading interferes with the return path, causing DNS responses to be dropped or misrouted.

**Workaround:** Explicitly set `forwardKubeDNSToHost: false` when using `bpf.masquerade: true`:

```bash
# Helm values for Cilium
cilium install \
  --set bpf.masquerade=true \
  --set forwardKubeDNSToHost=false \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup
```

Or in Helm values file:

```yaml
bpf:
  masquerade: true
forwardKubeDNSToHost: false
cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup
```

**Affected versions:** 1.9.0+ (any Talos version using Cilium with this combination)

**Status:** Workaround available. This is a Cilium configuration interaction, not a Talos bug. Setting `forwardKubeDNSToHost: false` is the recommended configuration for Talos + Cilium.

---

### GCP Internal Load Balancer Not Working with Cilium

**Symptom:** On Google Cloud Platform, services of type `LoadBalancer` using GCP Internal Load Balancers do not receive traffic. Health checks pass, but traffic sent to the load balancer IP is not delivered to pods.

**Cause:** GCP internal load balancers send traffic directly to node IPs with the destination set to the load balancer VIP. When Cilium replaces kube-proxy, it does not recognize this traffic as destined for a service unless `bpf.lbExternalClusterIP` is enabled.

**Workaround:** Enable `bpf.lbExternalClusterIP` in Cilium configuration:

```yaml
bpf:
  lbExternalClusterIP: true
```

Full Cilium install command for GCP + Talos:

```bash
cilium install \
  --set kubeProxyReplacement=true \
  --set bpf.masquerade=true \
  --set bpf.lbExternalClusterIP=true \
  --set forwardKubeDNSToHost=false \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup
```

**Affected versions:** 1.9.0+ (any Talos version running Cilium on GCP)

**Status:** Workaround available. This is a Cilium + GCP interaction, not specific to Talos.

---

### Pod Security Warnings in Cilium Connectivity Tests

**Symptom:** Running `cilium connectivity test` produces pod security admission warnings like:

```
Warning: would violate PodSecurity "restricted:latest":
allowPrivilegeEscalation != false, unrestricted capabilities, ...
```

Some connectivity test pods fail to schedule on clusters with `restricted` Pod Security Standards enforced.

**Cause:** Cilium connectivity test pods require elevated privileges (NET_ADMIN, NET_RAW, SYS_ADMIN capabilities) to perform network diagnostics. Kubernetes Pod Security Admission rejects or warns about these pods when the namespace does not have the `privileged` label.

**Workaround:** Label the Cilium test namespace with the `privileged` Pod Security Standard before running tests:

```bash
# If using the default cilium-test namespace:
kubectl create namespace cilium-test 2>/dev/null || true
kubectl label namespace cilium-test \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite

# Then run connectivity tests:
cilium connectivity test
```

If you already ran the test and it created the namespace without the label:

```bash
kubectl label namespace cilium-test \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
cilium connectivity test
```

**Affected versions:** 1.9.0+ (any Talos version with Kubernetes 1.25+ Pod Security Admission)

**Status:** Workaround available. This is expected behavior — Cilium tests require privileges that conflict with restricted Pod Security Standards.
