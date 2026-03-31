# Talos-Native Metrics

Talos Linux exposes Prometheus-format metrics from its controller runtime and from etcd. These endpoints are available on every node without deploying any additional agents.

## Metrics Endpoints

| Endpoint | Port | Path | Available On | Description |
|---|---|---|---|---|
| Controller Runtime | 2080 | /metrics | All nodes | Talos controller-runtime metrics (machined, apid, trustd, etc.) |
| etcd | 2381 | /metrics | Control plane nodes | etcd server, client, and disk metrics |

These endpoints serve standard Prometheus exposition format over HTTP (no TLS required for the metrics ports).

### Verify Endpoints

```bash
# Controller runtime metrics from a node
curl -s http://10.0.0.11:2080/metrics | head -20

# etcd metrics from a control plane node
curl -s http://10.0.0.11:2381/metrics | head -20

# Via talosctl (useful when nodes are not directly reachable)
talosctl -n 10.0.0.11 get machinestatuses
```

## Controller Runtime Metrics

The Talos controller runtime exposes metrics about internal controllers. Key metrics:

| Metric | Type | Description |
|---|---|---|
| `controller_runtime_reconcile_total` | Counter | Total reconciliation attempts per controller |
| `controller_runtime_reconcile_errors_total` | Counter | Failed reconciliation attempts per controller |
| `controller_runtime_reconcile_duration_seconds` | Histogram | Time spent in each reconciliation |
| `controller_runtime_queue_length` | Gauge | Current queue depth per controller |
| `go_goroutines` | Gauge | Number of goroutines in the Talos process |
| `go_memstats_alloc_bytes` | Gauge | Bytes allocated and in use |
| `process_cpu_seconds_total` | Counter | Total CPU time consumed by the Talos process |
| `process_resident_memory_bytes` | Gauge | Resident memory size |

### Useful PromQL Queries for Controller Runtime

```promql
# Reconciliation error rate per controller
sum by (controller) (rate(controller_runtime_reconcile_errors_total[5m]))

# Controllers with the highest reconciliation latency (p99)
histogram_quantile(0.99, sum by (controller, le) (rate(controller_runtime_reconcile_duration_seconds_bucket[5m])))

# Queue depth — controllers falling behind
controller_runtime_queue_length > 0

# Total reconciliations per second across all controllers
sum(rate(controller_runtime_reconcile_total[5m]))

# Memory usage of Talos process
process_resident_memory_bytes{job="talos-controller-runtime"}
```

## etcd Metrics

Talos runs etcd as a Talos service (not a static pod). The metrics endpoint is on port 2381. Key metrics:

| Metric | Type | Description |
|---|---|---|
| `etcd_server_has_leader` | Gauge | 1 if the member has a leader, 0 otherwise |
| `etcd_server_leader_changes_seen_total` | Counter | Leader elections observed |
| `etcd_mvcc_db_total_size_in_bytes` | Gauge | Total etcd database size |
| `etcd_mvcc_db_total_size_in_use_in_bytes` | Gauge | Used portion of etcd database |
| `etcd_disk_wal_fsync_duration_seconds` | Histogram | WAL fsync latency |
| `etcd_disk_backend_commit_duration_seconds` | Histogram | Backend commit latency |
| `etcd_network_peer_round_trip_time_seconds` | Histogram | Round-trip time between etcd peers |
| `etcd_server_proposals_committed_total` | Counter | Total committed Raft proposals |
| `etcd_server_proposals_failed_total` | Counter | Total failed Raft proposals |
| `etcd_server_proposals_pending` | Gauge | Pending Raft proposals |
| `grpc_server_handled_total` | Counter | Total gRPC requests handled by etcd |

### Useful PromQL Queries for etcd

```promql
# Cluster health — all members should have a leader
etcd_server_has_leader

# Leader changes (should be rare in a stable cluster)
rate(etcd_server_leader_changes_seen_total[1h])

# Database size per member
etcd_mvcc_db_total_size_in_bytes

# Database fragmentation ratio
1 - (etcd_mvcc_db_total_size_in_use_in_bytes / etcd_mvcc_db_total_size_in_bytes)

# WAL fsync latency (p99) — should be well under 100ms
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))

# Backend commit latency (p99)
histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m]))

# Peer round-trip time (p99)
histogram_quantile(0.99, rate(etcd_network_peer_round_trip_time_seconds_bucket[5m]))

# Pending proposals (should be near 0)
etcd_server_proposals_pending

# Failed proposals rate
rate(etcd_server_proposals_failed_total[5m])

# gRPC request rate by method
sum by (grpc_method) (rate(grpc_server_handled_total[5m]))
```

## Prometheus Scrape Configuration

If using kube-prometheus-stack, add these scrape jobs in the Helm values (see prometheus.md). For standalone Prometheus, add to `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: talos-controller-runtime
    static_configs:
      - targets:
          - 10.0.0.11:2080
          - 10.0.0.12:2080
          - 10.0.0.13:2080
          - 10.0.0.21:2080
          - 10.0.0.22:2080
        labels:
          cluster: talos-prod
    metrics_path: /metrics
    scrape_interval: 30s

  - job_name: talos-etcd
    static_configs:
      - targets:
          - 10.0.0.11:2381
          - 10.0.0.12:2381
          - 10.0.0.13:2381
        labels:
          cluster: talos-prod
    metrics_path: /metrics
    scrape_interval: 30s
```

### Dynamic Target Discovery with Kubernetes Endpoints

Instead of hardcoding node IPs, create headless Services and use Kubernetes service discovery:

```yaml
# Service to expose Talos controller-runtime metrics
apiVersion: v1
kind: Service
metadata:
  name: talos-controller-runtime
  namespace: kube-system
  labels:
    app: talos-controller-runtime
spec:
  clusterIP: None
  ports:
    - name: metrics
      port: 2080
      targetPort: 2080
---
apiVersion: v1
kind: Endpoints
metadata:
  name: talos-controller-runtime
  namespace: kube-system
  labels:
    app: talos-controller-runtime
subsets:
  - addresses:
      - ip: 10.0.0.11
      - ip: 10.0.0.12
      - ip: 10.0.0.13
      - ip: 10.0.0.21
      - ip: 10.0.0.22
    ports:
      - name: metrics
        port: 2080
---
# ServiceMonitor for Prometheus Operator
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: talos-controller-runtime
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      app: talos-controller-runtime
  endpoints:
    - port: metrics
      interval: 30s
```

Repeat for etcd:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: talos-etcd-metrics
  namespace: kube-system
  labels:
    app: talos-etcd-metrics
spec:
  clusterIP: None
  ports:
    - name: metrics
      port: 2381
      targetPort: 2381
---
apiVersion: v1
kind: Endpoints
metadata:
  name: talos-etcd-metrics
  namespace: kube-system
  labels:
    app: talos-etcd-metrics
subsets:
  - addresses:
      - ip: 10.0.0.11
      - ip: 10.0.0.12
      - ip: 10.0.0.13
    ports:
      - name: metrics
        port: 2381
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: talos-etcd-metrics
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      app: talos-etcd-metrics
  endpoints:
    - port: metrics
      interval: 30s
```

## Custom Grafana Dashboards

### Talos Controller Runtime Dashboard

Import via ConfigMap for automatic sidecar pickup:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-talos-runtime
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  talos-controller-runtime.json: |
    {
      "annotations": { "list": [] },
      "editable": true,
      "fiscalYearStartMonth": 0,
      "graphTooltip": 1,
      "id": null,
      "links": [],
      "panels": [
        {
          "title": "Reconciliation Rate by Controller",
          "type": "timeseries",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            {
              "expr": "sum by (controller) (rate(controller_runtime_reconcile_total{job=\"talos-controller-runtime\"}[5m]))",
              "legendFormat": "{{ controller }}"
            }
          ],
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 }
        },
        {
          "title": "Reconciliation Errors by Controller",
          "type": "timeseries",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            {
              "expr": "sum by (controller) (rate(controller_runtime_reconcile_errors_total{job=\"talos-controller-runtime\"}[5m]))",
              "legendFormat": "{{ controller }}"
            }
          ],
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 }
        },
        {
          "title": "Reconciliation Latency p99",
          "type": "timeseries",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            {
              "expr": "histogram_quantile(0.99, sum by (controller, le) (rate(controller_runtime_reconcile_duration_seconds_bucket{job=\"talos-controller-runtime\"}[5m])))",
              "legendFormat": "{{ controller }}"
            }
          ],
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 8 }
        },
        {
          "title": "Queue Length",
          "type": "timeseries",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            {
              "expr": "controller_runtime_queue_length{job=\"talos-controller-runtime\"}",
              "legendFormat": "{{ controller }}"
            }
          ],
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 8 }
        },
        {
          "title": "Process Memory",
          "type": "stat",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            {
              "expr": "process_resident_memory_bytes{job=\"talos-controller-runtime\"}",
              "legendFormat": "{{ instance }}"
            }
          ],
          "fieldConfig": { "defaults": { "unit": "bytes" } },
          "gridPos": { "h": 4, "w": 24, "x": 0, "y": 16 }
        }
      ],
      "schemaVersion": 39,
      "tags": ["talos"],
      "templating": { "list": [] },
      "time": { "from": "now-1h", "to": "now" },
      "title": "Talos Controller Runtime",
      "uid": "talos-controller-runtime"
    }
```

### Talos etcd Dashboard

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-talos-etcd
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  talos-etcd.json: |
    {
      "annotations": { "list": [] },
      "editable": true,
      "graphTooltip": 1,
      "panels": [
        {
          "title": "etcd Has Leader",
          "type": "stat",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            { "expr": "etcd_server_has_leader{job=\"talos-etcd\"}", "legendFormat": "{{ instance }}" }
          ],
          "fieldConfig": { "defaults": { "mappings": [{ "type": "value", "options": { "1": { "text": "YES", "color": "green" }, "0": { "text": "NO", "color": "red" } } }] } },
          "gridPos": { "h": 4, "w": 8, "x": 0, "y": 0 }
        },
        {
          "title": "Leader Changes",
          "type": "stat",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            { "expr": "increase(etcd_server_leader_changes_seen_total{job=\"talos-etcd\"}[1h])", "legendFormat": "{{ instance }}" }
          ],
          "gridPos": { "h": 4, "w": 8, "x": 8, "y": 0 }
        },
        {
          "title": "DB Size",
          "type": "stat",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            { "expr": "etcd_mvcc_db_total_size_in_bytes{job=\"talos-etcd\"}", "legendFormat": "{{ instance }}" }
          ],
          "fieldConfig": { "defaults": { "unit": "bytes" } },
          "gridPos": { "h": 4, "w": 8, "x": 16, "y": 0 }
        },
        {
          "title": "WAL Fsync Latency p99",
          "type": "timeseries",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            { "expr": "histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket{job=\"talos-etcd\"}[5m]))", "legendFormat": "{{ instance }}" }
          ],
          "fieldConfig": { "defaults": { "unit": "s" } },
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 4 }
        },
        {
          "title": "Backend Commit Latency p99",
          "type": "timeseries",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            { "expr": "histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket{job=\"talos-etcd\"}[5m]))", "legendFormat": "{{ instance }}" }
          ],
          "fieldConfig": { "defaults": { "unit": "s" } },
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 4 }
        },
        {
          "title": "Peer Round Trip Time p99",
          "type": "timeseries",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            { "expr": "histogram_quantile(0.99, rate(etcd_network_peer_round_trip_time_seconds_bucket{job=\"talos-etcd\"}[5m]))", "legendFormat": "{{ instance }}" }
          ],
          "fieldConfig": { "defaults": { "unit": "s" } },
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 12 }
        },
        {
          "title": "Pending Proposals",
          "type": "timeseries",
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            { "expr": "etcd_server_proposals_pending{job=\"talos-etcd\"}", "legendFormat": "{{ instance }}" }
          ],
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 12 }
        }
      ],
      "schemaVersion": 39,
      "tags": ["talos", "etcd"],
      "time": { "from": "now-1h", "to": "now" },
      "title": "Talos etcd",
      "uid": "talos-etcd"
    }
```

Apply both dashboards:

```bash
kubectl apply -f grafana-dashboard-talos-runtime.yaml
kubectl apply -f grafana-dashboard-talos-etcd.yaml
```

## talosctl dashboard Reference

For quick interactive diagnostics without the full Prometheus stack:

```bash
# Single node
talosctl dashboard --nodes 10.0.0.11

# Multiple nodes (use Tab to switch)
talosctl dashboard --nodes 10.0.0.11,10.0.0.12,10.0.0.13

# Keyboard shortcuts in the dashboard:
# Tab     — switch between nodes
# q       — quit
# 1-5     — switch between dashboard pages (summary, CPU, memory, network, disk)
```

The dashboard pages:

| Page | Key | Shows |
|---|---|---|
| Summary | 1 | CPU, memory, disk, network overview |
| CPU | 2 | Per-core CPU usage, load average |
| Memory | 3 | Memory breakdown (used, cached, buffers, available) |
| Network | 4 | Per-interface throughput and packet counts |
| Disk | 5 | Per-device IOPS and throughput |

## Operational Notes

- **Firewall rules**: ensure Prometheus pods can reach node IPs on ports 2080 and 2381. If using Cilium network policies, create a CiliumNetworkPolicy allowing egress from the monitoring namespace to these ports.
- **Node IP changes**: if nodes are provisioned dynamically, update the Endpoints resources or switch to a discovery mechanism that tracks node IPs automatically.
- **etcd defragmentation**: if the etcd database grows significantly (visible via `etcd_mvcc_db_total_size_in_bytes`), run defragmentation: `talosctl etcd defrag --nodes 10.0.0.11`.
