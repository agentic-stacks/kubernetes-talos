# Platform — Observability

Observability strategy for Talos Linux clusters. Talos is an immutable, API-driven OS with no SSH, no shell, and no systemd journal. Every observability approach must account for these constraints. This skill covers metrics, logging, tracing, and Talos-native monitoring tooling.

---

## 1. Talos Observability Model

Talos Linux exposes system telemetry through its API rather than traditional Linux interfaces:

- **No journalctl** — Talos does not run systemd. Logs are streamed through the Talos API and collected via `talosctl logs` or the gRPC streaming endpoint.
- **No node-level agents via SSH** — monitoring agents must run as Kubernetes workloads (DaemonSets) or scrape Talos API endpoints from within the cluster.
- **Metrics via API** — Talos exposes Prometheus-format metrics at `:2080/metrics` on each node for the Talos controller runtime, and at `:2381/metrics` for etcd on control plane nodes.
- **talosctl dashboard** — a built-in TUI that displays real-time CPU, memory, network, disk, and process information for any node, useful for interactive debugging without deploying a full stack.

### talosctl dashboard

The dashboard is a terminal UI for live node inspection:

```bash
# Connect to a single node
talosctl dashboard --nodes 10.0.0.11

# Connect to multiple nodes (tab between them)
talosctl dashboard --nodes 10.0.0.11,10.0.0.12,10.0.0.13
```

The dashboard shows:
- CPU usage per core
- Memory usage (used, cached, buffers)
- Network throughput per interface
- Disk I/O per device
- Running processes with resource consumption
- Talos services and their states

This is the fastest way to check node health without deploying any observability stack.

---

## 2. Recommended Stack

The recommended production observability stack for Talos clusters:

| Layer | Tool | Purpose |
|---|---|---|
| Metrics | kube-prometheus-stack | Prometheus + Grafana + AlertManager, with pre-built dashboards and recording rules |
| Logging | Loki + Grafana Alloy | Log aggregation with LogQL, lightweight indexing |
| Tracing | OpenTelemetry Collector + Tempo | Distributed tracing with Grafana integration |
| Talos-native | talosctl + custom scrape configs | Node-level metrics from the Talos API |

This stack runs entirely within Kubernetes. No host-level agents or SSH access needed.

### Why This Stack

- **kube-prometheus-stack** bundles Prometheus Operator, Grafana, AlertManager, node-exporter, kube-state-metrics, and default recording/alerting rules. It is the most widely deployed Kubernetes monitoring solution and has explicit support for etcd, kubelet, and API server metrics.
- **Loki** is chosen over Elasticsearch for its lower resource footprint and native Grafana integration. It indexes labels only (not full text), which keeps storage costs manageable.
- **OpenTelemetry Collector** provides a vendor-neutral pipeline for traces (and optionally metrics/logs), avoiding lock-in to any specific backend.
- **Tempo** is the recommended trace backend for Grafana-centric stacks. Jaeger is also supported.

---

## 3. Talos-Specific Considerations

### Machine Config for Metrics Endpoints

Talos exposes controller-runtime metrics on port 2080 and etcd metrics on port 2381 by default. No machine config change is needed to enable these. However, you must ensure network policies allow Prometheus to reach these ports on node IPs.

### Kernel Logs

Talos kernel logs (dmesg) are available via the API:

```bash
talosctl dmesg --nodes 10.0.0.11
talosctl dmesg --nodes 10.0.0.11 --follow
```

### Talos Service Logs

Each Talos service (apid, machined, trustd, etcd, kubelet) streams logs through the API:

```bash
# Specific service logs
talosctl logs machined --nodes 10.0.0.11
talosctl logs etcd --nodes 10.0.0.11
talosctl logs kubelet --nodes 10.0.0.11

# Follow mode
talosctl logs kubelet --nodes 10.0.0.11 --follow
```

### Log Collection Strategy

Talos does not write logs to files on disk. For centralized logging, you have two options:

1. **Container logs via kubelet** — standard Kubernetes pod logs collected by Alloy/Promtail DaemonSets reading from `/var/log/pods`. This covers all workload logs.
2. **Talos service logs via API** — for Talos-level service logs (machined, etcd, apid), use a sidecar or CronJob that calls the Talos gRPC API and forwards to Loki. Alternatively, configure Talos to send logs to an external endpoint.

### Talos Log Sink (Machine Config)

Talos can forward its own service logs to an external receiver:

```yaml
machine:
  logging:
    destinations:
      - endpoint: "udp://10.0.0.50:5514"
        format: json_lines
      - endpoint: "tcp://10.0.0.50:5514"
        format: json_lines
```

This sends Talos-level logs (machined, etcd, apid, kubelet) to a syslog or JSON receiver. Configure an Alloy or Vector instance to receive these and forward to Loki.

---

## 4. Resource Requirements

Baseline sizing for a 5-node cluster with moderate workloads:

| Component | CPU Request | Memory Request | Storage |
|---|---|---|---|
| Prometheus | 500m | 2Gi | 50Gi PV |
| Grafana | 100m | 256Mi | 1Gi PV |
| AlertManager | 50m | 128Mi | 1Gi PV |
| Loki (write) | 250m | 512Mi | 10Gi PV |
| Loki (read) | 250m | 512Mi | — |
| Alloy (DaemonSet) | 100m/node | 128Mi/node | — |
| OTEL Collector | 200m | 256Mi | — |
| Tempo | 250m | 512Mi | 10Gi PV |

Total overhead: approximately 2–3 CPU cores and 5–6 GiB RAM. Scale horizontally as cluster size grows.

---

## 5. Deployment Order

Install in this sequence to satisfy dependencies:

1. **Storage** — ensure a default StorageClass exists for PVCs (see `deploy/storage`)
2. **kube-prometheus-stack** — deploys Prometheus, Grafana, AlertManager, node-exporter, kube-state-metrics
3. **Loki + Alloy** — deploys log aggregation and collection
4. **Talos metrics scrape config** — adds Prometheus scrape jobs for Talos-native endpoints
5. **OpenTelemetry + Tempo** — deploys tracing pipeline (optional, add when applications emit traces)

---

## 6. Skill Index

| Topic | Guide | Covers |
|---|---|---|
| Prometheus & Grafana | [prometheus.md](./prometheus.md) | kube-prometheus-stack, dashboards, alerts, retention |
| Loki & Log Collection | [loki.md](./loki.md) | Loki Helm, Alloy DaemonSet, LogQL, Talos log strategy |
| Talos-Native Metrics | [talos-metrics.md](./talos-metrics.md) | Controller-runtime metrics, etcd metrics, custom dashboards |
| Tracing | [tracing.md](./tracing.md) | OTEL Collector, Tempo/Jaeger, auto-instrumentation |

---

## Next Step

After deploying the observability stack, connect alerts to your incident management workflow:

```
platform/observability → platform/security (audit logging)
```
