# Platform — Service Mesh: Selection Guide

## Do You Need a Service Mesh?

A service mesh adds a dedicated infrastructure layer for service-to-service communication. Before adopting one, answer these questions:

| Question | If Yes | If No |
|---|---|---|
| Do you need mutual TLS between all services? | Mesh adds this transparently | Use NetworkPolicy for L3/L4 isolation instead |
| Do you need L7 traffic management (retries, timeouts, circuit breaking)? | Mesh provides this per-service | Application-level libraries or Gateway API may suffice |
| Do you need fine-grained traffic splitting (canary, A/B)? | Mesh makes this declarative | Use Gateway API or Argo Rollouts at the ingress layer |
| Do you need distributed tracing headers injected automatically? | Mesh sidecars/eBPF can do this | Instrument applications directly with OpenTelemetry |
| Do you have fewer than 10 services? | Mesh overhead likely not justified | Mesh overhead likely not justified |
| Are you running multi-cluster? | Mesh simplifies cross-cluster auth and routing | Single-cluster NetworkPolicy may be enough |

**Rule of thumb**: If you only need mTLS and L3/L4 policy, Cilium's native mesh capabilities (no sidecar, eBPF-based) are the lowest-overhead option. Only adopt a full sidecar mesh (Istio, Linkerd) when you need L7 traffic management features that Cilium does not yet cover.

## Decision Matrix

| Feature | Cilium Mesh | Linkerd | Istio |
|---|---|---|---|
| **Architecture** | eBPF in-kernel, no sidecars | Rust micro-proxies (sidecar) | Envoy sidecars |
| **mTLS** | Mutual authentication (SPIFFE) | On by default | PeerAuthentication policy |
| **L7 policy** | CiliumNetworkPolicy + CiliumEnvoyConfig | HTTPRoute, ServiceProfile | VirtualService, AuthorizationPolicy |
| **Traffic splitting** | CiliumEnvoyConfig weighted clusters | TrafficSplit (SMI), HTTPRoute | VirtualService weight |
| **Observability** | Hubble (flows, DNS, HTTP, Kafka) | Linkerd Viz (golden metrics) | Kiali, Prometheus, Jaeger |
| **Resource overhead** | Lowest — no sidecar per pod | Low — Rust proxy ~10 MB per pod | Highest — Envoy ~50 MB per pod |
| **Sidecar injection** | None required | Annotation-based | Namespace label |
| **Gateway API** | Native (if enabled in Cilium CNI) | Supported via policy CRDs | Supported (Istio gateway) |
| **Multi-cluster** | ClusterMesh | Linkerd multi-cluster | Istio multi-cluster |
| **Talos compatibility** | Excellent — native CNI integration | Good — no kernel module issues | Good — requires kernel param tuning |
| **Complexity** | Low (if already using Cilium CNI) | Low-Medium | High |

## Talos-Specific Considerations

Talos Linux is an immutable, API-driven OS. This affects service mesh deployment in several ways:

### No iptables manipulation from userspace

Talos does not allow pods to manipulate iptables rules via `NET_ADMIN` in the way traditional init containers expect. This affects:

- **Istio**: The `istio-init` init container uses `iptables` to redirect traffic. On Talos, use the **Istio CNI plugin** (`components.cni.enabled=true`) instead of init containers. The CNI plugin runs as a DaemonSet with host networking and configures iptables at the node level.
- **Linkerd**: The `linkerd-init` init container also uses `iptables`. Use the **Linkerd CNI plugin** (`--set cniEnabled=true`) to avoid this. Alternatively, Linkerd 2.14+ supports a `--set proxyInit.iptablesMode=nft` flag, but the CNI plugin is the recommended approach on Talos.
- **Cilium mesh**: No iptables manipulation needed — everything runs in eBPF.

### Kernel restrictions

Talos ships a locked-down kernel. Key points:

- **No kernel module loading** from workloads. Service meshes that attempt to load modules will fail silently or crash.
- **cgroup mounts** are managed by Talos. Set `cgroup.autoMount.enabled=false` and `cgroup.hostRoot=/sys/fs/cgroup` for Cilium.
- **Filesystem is read-only**. Meshes that write to `/etc` or `/var/lib` must use `emptyDir` volumes or Talos-approved paths.

### Recommended approach by scenario

| Scenario | Recommendation |
|---|---|
| Already using Cilium CNI | Enable Cilium mesh features — zero additional components |
| Need full L7 traffic management | Istio with CNI plugin, or Linkerd with CNI plugin |
| Want simplest mTLS with minimal overhead | Cilium mutual authentication |
| Resource-constrained edge clusters | Linkerd (Rust proxy, smallest footprint among sidecar meshes) |
| Feature-rich enterprise requirements | Istio (largest ecosystem, most configuration options) |

## Service Mesh Guides

- [Cilium Mesh](./cilium-mesh.md) — eBPF-based, no sidecars, native with Cilium CNI
- [Istio](./istio.md) — Envoy-based, feature-rich, CNI plugin required on Talos
- [Linkerd](./linkerd.md) — Rust-based, lightweight, mTLS by default
