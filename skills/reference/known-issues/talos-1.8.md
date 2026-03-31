# Known Issues — Talos 1.8.x

---

### Disk Encryption Key Slot Exhaustion on Repeated Upgrades

**Symptom:** After multiple Talos upgrades or config applies, the node fails to boot with disk encryption errors. LUKS reports that all key slots are consumed.

**Cause:** In Talos 1.8.0-1.8.1, each configuration apply or upgrade that touched disk encryption settings could add a new LUKS key slot without removing the old one. After 8 applies (the LUKS2 key slot limit), no new keys could be added.

**Workaround:** Upgrade to Talos 1.8.2+ where key slot management was fixed. If already stuck:

1. Boot from a Talos ISO in maintenance mode
2. Manually manage LUKS key slots to free space
3. Apply the config and upgrade to 1.8.2+

**Affected versions:** 1.8.0-1.8.1

**Status:** Fixed in 1.8.2.

---

### KubePrism Health Check Flapping on Slow Networks

**Symptom:** KubePrism (the in-process kube-apiserver load balancer on each node) intermittently marks healthy API server endpoints as unhealthy. kubectl commands from within pods sporadically fail with connection refused or timeout errors.

**Cause:** The default KubePrism health check interval and timeout were too aggressive for environments with higher network latency (e.g., cross-datacenter clusters, certain cloud VPCs). Healthy endpoints would be marked down during brief latency spikes.

**Workaround:** This was addressed with tunable health check parameters in Talos 1.8.1. If running 1.8.0, upgrade to 1.8.1+. For environments with known latency, verify KubePrism is not the source of intermittent API failures:

```bash
# Check KubePrism status
talosctl get kubeprismstatuses -n <node>

# Check KubePrism endpoints
talosctl get kubeprismendpoints -n <node>
```

**Affected versions:** 1.8.0

**Status:** Fixed in 1.8.1.

---

### Extension Services Restart Loop After Config Apply

**Symptom:** Custom extension services (installed via system extensions) enter a restart loop after `talosctl apply-config`. The service starts, stops, and restarts continuously. Standard Talos services (etcd, kubelet) are unaffected.

**Cause:** A race condition in the service supervisor caused extension services to receive a restart signal during config convergence, even when their configuration had not changed. The restart would trigger another convergence check, creating a loop.

**Workaround:** Upgrade to Talos 1.8.3+ where the service supervisor was fixed to skip restarts for unchanged services. Temporary mitigation for 1.8.0-1.8.2:

```bash
# Manually restart the affected extension service after apply-config settles
talosctl service <extension-name> restart -n <node>
```

**Affected versions:** 1.8.0-1.8.2

**Status:** Fixed in 1.8.3.

---

### Flannel Pods CrashLoopBackOff After Kubernetes 1.31 Upgrade

**Symptom:** After upgrading Kubernetes to 1.31 on Talos 1.8.x, Flannel pods enter CrashLoopBackOff. Logs show errors related to the deprecated `--iptables-mode` flag or CNI binary version mismatches.

**Cause:** The bundled Flannel version in early Talos 1.8.x releases was not compatible with Kubernetes 1.31's updated CNI specification. The Flannel DaemonSet manifest referenced flags removed in newer kube-proxy versions.

**Workaround:** Upgrade Flannel independently of Talos:

```bash
# If using the Talos-managed Flannel (default CNI), upgrade Talos to 1.8.4+
# where the bundled Flannel version was updated.

# If managing Flannel yourself via Helm:
helm upgrade flannel flannel/flannel -n kube-flannel \
  --set podCidr="10.244.0.0/16"
```

Alternatively, this is a good opportunity to migrate to Cilium (requires cluster rebuild since CNI is a bootstrap-time decision).

**Affected versions:** 1.8.0-1.8.3 with Kubernetes 1.31

**Status:** Fixed in 1.8.4.

---

### Predictable Interface Names Not Available (eudev)

**Symptom:** Network interfaces always use kernel-assigned names (`eth0`, `eth1`) regardless of hardware topology. Users expecting `enp0s3`-style names cannot use them.

**Cause:** Talos 1.8.x uses `eudev` which does not implement predictable interface naming. This is not a bug but a design choice in 1.8.x.

**Workaround:** Use `deviceSelector` with `hardwareAddr` or `busPath` to identify interfaces reliably, rather than depending on predictable names. Or upgrade to Talos 1.9.x where `systemd-udevd` provides predictable interface naming.

```yaml
machine:
  network:
    interfaces:
      - deviceSelector:
            hardwareAddr: "aa:bb:cc:*"   # Wildcard match supported
        dhcp: true
```

**Affected versions:** All 1.8.x

**Status:** By design. Resolved in Talos 1.9.0 with the switch to systemd-udevd.
