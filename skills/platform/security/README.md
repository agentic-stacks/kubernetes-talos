# Platform — Security

Security posture for Kubernetes clusters running on Talos Linux. Talos provides a hardened OS-level baseline that eliminates entire categories of attack. This skill covers what Talos gives you for free, what you must add at the Kubernetes layer, and how the security layers fit together.

---

## 1. Talos Security Baseline

Talos Linux is purpose-built for Kubernetes and ships with security defaults that cannot be weakened at runtime:

| Property | Detail |
|---|---|
| **No shell access** | No SSH, no console login, no interactive shell. All node operations go through `talosctl` and the Talos API. |
| **Immutable root filesystem** | The root filesystem is read-only and verified at boot. No runtime modification of OS binaries or libraries. |
| **Mutual TLS everywhere** | All Talos API communication (talosctl, inter-node) uses mTLS. Certificates are generated during `talosctl gen config`. |
| **Secretbox encryption** | Kubernetes Secrets are encrypted at rest using Secretbox (XSalsa20-Poly1305) by default. No additional etcd encryption configuration needed. |
| **Minimal attack surface** | No package manager, no extra services, no userland tools. Only Kubernetes-required components run. |
| **Secure boot support** | UKI (Unified Kernel Image) support for UEFI Secure Boot chains. |
| **Kernel hardening** | Talos applies kernel hardening parameters: `kernel.kptr_restrict=1`, `kernel.dmesg_restrict=1`, `kernel.perf_event_paranoid=3`, among others. |

### Verify Talos Security Settings

```bash
# Check that Secretbox encryption is active
talosctl get machineconfig -o yaml --nodes 10.0.0.21 | grep -A5 aescbcEncryption

# Verify mTLS certificates
talosctl config info

# Check running extensions (should be minimal)
talosctl get extensions --nodes 10.0.0.21

# Verify kernel parameters
talosctl read /proc/sys/kernel/kptr_restrict --nodes 10.0.0.21
```

---

## 2. What Talos Does NOT Cover

Talos secures the OS layer. The Kubernetes layer requires additional hardening:

| Gap | Mitigation | Skill |
|---|---|---|
| No Pod Security enforcement | Pod Security Standards via admission controller or Kyverno/Gatekeeper | [pod-security.md](./pod-security.md) |
| No secrets management beyond etcd encryption | Sealed Secrets, External Secrets Operator, or Vault | [secrets-management.md](./secrets-management.md) |
| Default RBAC is permissive | Least-privilege Roles, scoped ServiceAccounts, audit logging | [rbac.md](./rbac.md) |
| No network segmentation by default | NetworkPolicy, CiliumNetworkPolicy, or Calico GlobalNetworkPolicy | [network-policy.md](./network-policy.md) |
| No image scanning | Trivy, Grype, or admission-time image verification | Out of scope (CI/CD layer) |
| No runtime threat detection | Falco or Tetragon for runtime security monitoring | Out of scope (observability layer) |

---

## 3. CIS Benchmark Alignment

Talos Linux aligns with the CIS Kubernetes Benchmark by default in several areas:

### Passed by Default (Talos handles these)

- **1.1.x** — API server file permissions: not applicable (no host shell access)
- **1.2.6** — `--kubelet-certificate-authority` is set via Talos machine config
- **1.2.16** — Admission plugins `NodeRestriction` enabled by default
- **1.2.22** — Audit logging can be enabled via machine config extra args
- **4.1.x** — Kubelet file permissions: not applicable (immutable OS)
- **4.2.1** — Anonymous auth disabled by default on kubelet
- **5.1.5** — Default ServiceAccount tokens not auto-mounted (K8s 1.24+)

### Requires Manual Configuration

- **1.2.22–1.2.25** — Audit logging: must configure audit policy via `extraArgs` and `extraVolumes`
- **5.1.x** — RBAC least-privilege: must create scoped Roles and RoleBindings
- **5.2.x** — Pod Security Standards: must configure PodSecurity admission or policy engine
- **5.3.x** — Network Policies: must deploy and enforce per-namespace
- **5.4.x** — Secrets management: must use external secrets solution for production

### Enable Audit Logging

```yaml
# Machine config patch for API server audit logging
cluster:
  apiServer:
    extraArgs:
      audit-policy-file: /etc/kubernetes/audit-policy.yaml
      audit-log-path: /var/log/kubernetes/audit.log
      audit-log-maxage: "30"
      audit-log-maxbackup: "10"
      audit-log-maxsize: "100"
    extraVolumes:
      - name: audit-policy
        hostPath: /var/etc/kubernetes/audit-policy.yaml
        mountPath: /etc/kubernetes/audit-policy.yaml
        readOnly: true
      - name: audit-log
        hostPath: /var/log/kubernetes
        mountPath: /var/log/kubernetes
```

---

## 4. Security Layers Overview

```
┌─────────────────────────────────────────────────┐
│  Layer 5: Application Security                  │
│  Image scanning, SBOM, supply chain             │
├─────────────────────────────────────────────────┤
│  Layer 4: Runtime Security                      │
│  Falco, Tetragon, runtime threat detection      │
├─────────────────────────────────────────────────┤
│  Layer 3: Policy Enforcement                    │
│  Pod Security, Kyverno/Gatekeeper, RBAC         │
│  → pod-security.md, rbac.md                     │
├─────────────────────────────────────────────────┤
│  Layer 2: Network Security                      │
│  NetworkPolicy, mTLS (service mesh), DNS policy │
│  → network-policy.md                            │
├─────────────────────────────────────────────────┤
│  Layer 1: Secrets & Data Protection             │
│  Secretbox (etcd), Sealed Secrets, Vault, ESO   │
│  → secrets-management.md                        │
├─────────────────────────────────────────────────┤
│  Layer 0: OS & Infrastructure (Talos)           │
│  Immutable OS, mTLS API, no shell, Secure Boot  │
│  → Provided by Talos Linux                      │
└─────────────────────────────────────────────────┘
```

This skill covers Layers 1–3. Layers 4–5 belong to the observability and CI/CD stacks respectively.

---

## 5. Security Skill Index

| Topic | Guide | Covers |
|---|---|---|
| Pod Security | [pod-security.md](./pod-security.md) | PSS, PodSecurity admission, Kyverno, OPA Gatekeeper |
| Secrets Management | [secrets-management.md](./secrets-management.md) | Sealed Secrets, External Secrets Operator, Vault |
| RBAC | [rbac.md](./rbac.md) | Roles, bindings, ServiceAccounts, audit logging |
| Network Policy | [network-policy.md](./network-policy.md) | Default-deny, namespace isolation, Cilium L7, Calico |

---

## Next Step

Start with Pod Security Standards (Layer 3), then layer in secrets management and network policies:

```
platform/security/pod-security → platform/security/secrets-management → platform/security/rbac → platform/security/network-policy
```
