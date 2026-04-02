# Decision Guide — HA Endpoint Strategy

## Context

The control plane endpoint is the address that all nodes (kubelet) and external clients (kubectl) use to reach the Kubernetes API server. In a multi-control-plane cluster, this endpoint must be highly available — if one API server goes down, traffic must route to a healthy one.

**When to decide:** Before running `talosctl gen config`. The endpoint is baked into machine configs as `cluster.controlPlane.endpoint`.

**Can it be changed later?** Yes, but with effort. Changing the endpoint requires patching the machine config on every node and updating the kubeconfig. Plan this decision carefully.

**Talos-specific note:** Talos configures `cluster.controlPlane.endpoint` in the machine config. This value is used by kubelet, KubePrism (local API proxy), and certificate generation (certSANs). It is NOT used for Talos API access — `talosctl` uses individual node IPs.

---

## Options

### Talos VIP (Built-in Virtual IP)

Talos has a built-in Virtual IP implementation that floats a shared IP address across control plane nodes using layer-2 ARP/NDP announcements.

**How it works:**
1. Configure a VIP address in the machine config of all control plane nodes
2. One CP node claims the VIP and responds to ARP requests
3. If that node fails, another CP node takes over the VIP within seconds
4. No external infrastructure required

**Configuration:**

```yaml
machine:
  network:
    interfaces:
      - interface: eth0   # or use deviceSelector
        dhcp: true
        vip:
          ip: 10.0.0.100   # The shared VIP address
```

The VIP must also be in `certSANs` and used as the endpoint:

```yaml
machine:
  certSANs:
    - 10.0.0.100

cluster:
  controlPlane:
    endpoint: https://10.0.0.100:6443
```

**Strengths:**
- Zero external dependencies — built into Talos
- Simple configuration — just add the `vip` block
- Fast failover (seconds)
- No additional cost

**Weaknesses:**
- **Layer-2 only** — all CP nodes must be on the same L2 broadcast domain (same VLAN/subnet)
- Does NOT work across subnets, VPCs, availability zones, or cloud environments without L2 adjacency
- Does NOT work on most cloud platforms (AWS, GCP, Azure block gratuitous ARP)
- Single VIP = single point of entry (no load balancing, just failover)
- VIP failover may cause brief connection interruptions

**Best for:** Bare-metal, Proxmox, homelab, single-subnet on-prem environments.

### External Load Balancer (HAProxy, Nginx, keepalived)

A dedicated load balancer (physical, VM, or container) that sits in front of the control plane nodes and distributes traffic.

**How it works:**
1. Deploy a load balancer outside the Kubernetes cluster
2. Configure it to health-check and forward TCP 6443 to all CP nodes
3. Use the load balancer's IP/hostname as the control plane endpoint

**Example HAProxy configuration:**

```
frontend kubernetes-api
    bind *:6443
    mode tcp
    default_backend kubernetes-cp

backend kubernetes-cp
    mode tcp
    balance roundrobin
    option tcp-check
    server cp1 10.0.0.11:6443 check inter 5s fall 3 rise 2
    server cp2 10.0.0.12:6443 check inter 5s fall 3 rise 2
    server cp3 10.0.0.13:6443 check inter 5s fall 3 rise 2
```

**Strengths:**
- Works across subnets, VLANs, and cloud environments
- True load balancing (not just failover)
- Can add TLS termination, rate limiting, access logging
- Health checking removes unhealthy backends automatically
- Battle-tested in enterprise environments

**Weaknesses:**
- Requires infrastructure outside the Kubernetes cluster
- The load balancer itself must be HA (dual HAProxy with keepalived, or a managed LB)
- Additional maintenance burden and failure domain
- Cost (hardware, VM, or managed service)

**Best for:** On-prem production, environments spanning multiple subnets, any deployment requiring true load balancing.

### DNS Round-Robin

Use DNS with multiple A records pointing to all control plane node IPs. Client-side resolution distributes requests across CP nodes.

**How it works:**
1. Create a DNS A record (e.g., `k8s.example.com`) with entries for each CP node IP
2. Set a low TTL (30-60 seconds) for faster failover
3. Use the DNS name as the control plane endpoint

**DNS configuration:**

```
k8s.example.com.  30  IN  A  10.0.0.11
k8s.example.com.  30  IN  A  10.0.0.12
k8s.example.com.  30  IN  A  10.0.0.13
```

**Talos config:**

```yaml
machine:
  certSANs:
    - k8s.example.com
    - 10.0.0.11
    - 10.0.0.12
    - 10.0.0.13

cluster:
  controlPlane:
    endpoint: https://k8s.example.com:6443
```

**Strengths:**
- No additional infrastructure (if you already have DNS)
- Works across subnets and availability zones
- Simple to understand and configure

**Weaknesses:**
- No health checking — DNS will return IPs of failed nodes until records are updated
- TTL-based failover is slow (30-60 seconds minimum, often minutes due to caching)
- Client DNS caching may ignore TTL
- Not true load balancing — distribution depends on client resolver behavior
- Requires DNS infrastructure (but most environments have this)

**Best for:** Simple setups where DNS exists and brief outages during failover are acceptable. Not recommended for production without a fallback.

### Cloud Load Balancer (AWS NLB/ALB, GCP LB, Azure LB)

Managed load balancer service from the cloud provider, purpose-built for this use case.

**How it works:**
1. Create a TCP load balancer in the cloud provider
2. Add all CP node instances as backends on port 6443
3. Configure health checks (TCP or HTTPS on 6443)
4. Use the load balancer's DNS name or IP as the endpoint

**Example (AWS NLB via Terraform):**

```hcl
resource "aws_lb" "k8s_api" {
  name               = "k8s-api"
  internal           = true
  load_balancer_type = "network"
  subnets            = var.subnet_ids
}

resource "aws_lb_target_group" "k8s_api" {
  port     = 6443
  protocol = "TCP"
  vpc_id   = var.vpc_id

  health_check {
    protocol = "TCP"
    port     = 6443
  }
}

resource "aws_lb_listener" "k8s_api" {
  load_balancer_arn = aws_lb.k8s_api.arn
  port              = 6443
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.k8s_api.arn
  }
}
```

**Strengths:**
- Fully managed — no maintenance, automatic HA
- Health checking and automatic backend removal
- Works across availability zones natively
- Integrates with cloud IAM, monitoring, and logging
- High throughput and low latency

**Weaknesses:**
- Only available in cloud environments
- Cost (hourly + per-GB charges)
- Vendor lock-in
- Adds dependency on cloud provider control plane

**Best for:** Any cluster running on a cloud platform. This is the standard approach for cloud deployments.

---

## Comparison Table

| Feature | Talos VIP | External LB | DNS Round-Robin | Cloud LB |
|---------|-----------|-------------|-----------------|----------|
| **Infrastructure needed** | None | LB VM/appliance | DNS server | Cloud account |
| **Layer-2 required** | Yes | No | No | No |
| **Cross-subnet** | No | Yes | Yes | Yes |
| **Cross-AZ / cloud** | No | Yes | Yes | Yes (native) |
| **Health checking** | ARP-based (alive/dead) | TCP/HTTP health checks | None | TCP/HTTP health checks |
| **Failover time** | 2-5 seconds | 5-15 seconds | 30-300 seconds (TTL) | 5-15 seconds |
| **Load balancing** | No (failover only) | Yes | DNS-level only | Yes |
| **Additional cost** | None | LB infrastructure | None | Cloud LB pricing |
| **Operational complexity** | Very low | Medium-High | Low | Very low |
| **Single point of failure** | No (distributed) | LB itself (need HA LB) | DNS server | Cloud provider |
| **Bare-metal** | Yes | Yes | Yes | No |
| **Proxmox / homelab** | Yes | Yes | Yes | No |
| **AWS / GCP / Azure** | No | Yes | Yes | Yes (recommended) |
| **Hetzner Cloud** | Yes (same network) | Yes | Yes | Yes (Hetzner LB) |

---

## Recommendation by Platform

| Platform | Primary | Fallback | Notes |
|----------|---------|----------|-------|
| **Bare-metal (single subnet)** | Talos VIP | External LB | VIP is simplest when L2 is available |
| **Bare-metal (multi-subnet)** | External LB (HAProxy) | DNS round-robin | Need L3-capable solution |
| **Proxmox** | Talos VIP | External LB | VMs on same bridge = L2 adjacency |
| **AWS** | NLB (internal) | DNS round-robin | NLB is standard for EKS-like setups |
| **GCP** | Internal TCP LB | DNS round-robin | Native health checking |
| **Azure** | Azure Internal LB | DNS round-robin | Native health checking |
| **Hetzner Cloud** | Hetzner LB or Talos VIP | DNS round-robin | Both work; LB is more robust |
| **Homelab** | Talos VIP | DNS round-robin | VIP requires zero extra infra |
| **Edge / single-node** | N/A | N/A | Single CP node = no HA needed |

---

## Migration Path

Changing the control plane endpoint after cluster creation:

1. **Set up the new endpoint** (new VIP, LB, or DNS) pointing to existing CP nodes
2. **Add the new endpoint to certSANs** on all control plane nodes:
   ```bash
   talosctl patch mc -n <cp-node> --patch '[
     {"op": "add", "path": "/machine/certSANs/-", "value": "<new-endpoint>"}
   ]'
   ```
3. **Update the controlPlane endpoint** in machine config on all nodes:
   ```bash
   talosctl patch mc -n <node> --patch '[
     {"op": "replace", "path": "/cluster/controlPlane/endpoint", "value": "https://<new-endpoint>:6443"}
   ]'
   ```
4. **Regenerate kubeconfig** with the new endpoint:
   ```bash
   talosctl kubeconfig --nodes <cp-node> --force
   ```
5. **Verify** all nodes are healthy with the new endpoint:
   ```bash
   talosctl health --nodes <cp1>,<cp2>,<cp3>
   kubectl get nodes
   ```

**Key takeaway:** For bare-metal and homelab, start with Talos VIP. For cloud, use the cloud load balancer. Only reach for external LB or DNS when the simpler options do not fit your network topology.
