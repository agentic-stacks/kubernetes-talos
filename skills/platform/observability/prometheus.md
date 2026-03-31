# kube-prometheus-stack on Talos

kube-prometheus-stack deploys Prometheus Operator, Prometheus, Grafana, AlertManager, node-exporter, and kube-state-metrics via a single Helm chart. This guide covers Talos-specific values, ServiceMonitor configuration, alerting, and retention.

## Prerequisites

- A running Talos cluster with kubectl and Helm configured
- A default StorageClass for persistent volumes (see `deploy/storage`)
- Helm 3.x installed locally

## Installation

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Create a values file tailored for Talos:

```yaml
# prometheus-stack-values.yaml

# --- Prometheus ---
prometheus:
  prometheusSpec:
    retention: 15d
    retentionSize: "45GB"
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        memory: 4Gi
    # Scrape interval
    scrapeInterval: 30s
    evaluationInterval: 30s
    # Enable remote write if forwarding to Thanos, Mimir, etc.
    # remoteWrite:
    #   - url: "http://mimir-distributor.mimir:8080/api/v1/push"

    # Additional scrape configs for Talos-native metrics
    additionalScrapeConfigs:
      - job_name: talos-controller-runtime
        static_configs:
          - targets:
              - 10.0.0.11:2080
              - 10.0.0.12:2080
              - 10.0.0.13:2080
              - 10.0.0.21:2080
              - 10.0.0.22:2080
        metrics_path: /metrics
        scheme: http

      - job_name: talos-etcd
        static_configs:
          - targets:
              - 10.0.0.11:2381
              - 10.0.0.12:2381
              - 10.0.0.13:2381
        metrics_path: /metrics
        scheme: http

# --- Grafana ---
grafana:
  adminPassword: "changeme-use-secret-in-production"
  persistence:
    enabled: true
    size: 1Gi
  sidecar:
    dashboards:
      enabled: true
      searchNamespace: ALL
    datasources:
      enabled: true
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki-gateway.loki:80
      access: proxy
      jsonData:
        maxLines: 1000
    - name: Tempo
      type: tempo
      url: http://tempo-query-frontend.tempo:3100
      access: proxy
      jsonData:
        tracesToLogsV2:
          datasourceUid: loki
          filterByTraceID: true
        nodeGraph:
          enabled: true
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      memory: 512Mi

# --- AlertManager ---
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi
    resources:
      requests:
        cpu: 50m
        memory: 128Mi
      limits:
        memory: 256Mi
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: default
      routes:
        - receiver: critical-pagerduty
          match:
            severity: critical
          repeat_interval: 15m
        - receiver: warning-slack
          match:
            severity: warning
    receivers:
      - name: default
        slack_configs:
          - api_url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
            channel: "#alerts"
            title: '{{ .GroupLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
      - name: warning-slack
        slack_configs:
          - api_url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
            channel: "#alerts-warning"
            title: '{{ .GroupLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
      - name: critical-pagerduty
        pagerduty_configs:
          - service_key: "YOUR_PAGERDUTY_SERVICE_KEY"
            severity: critical

# --- node-exporter ---
nodeExporter:
  enabled: true

# --- kube-state-metrics ---
kubeStateMetrics:
  enabled: true

# --- Kubelet ServiceMonitor ---
kubelet:
  enabled: true
  serviceMonitor:
    metricRelabelings:
      # Drop high-cardinality kubelet metrics to reduce storage
      - sourceLabels: [__name__]
        regex: 'kubelet_runtime_operations_duration_seconds_bucket'
        action: drop

# --- etcd ServiceMonitor ---
kubeEtcd:
  enabled: true
  endpoints:
    - 10.0.0.11
    - 10.0.0.12
    - 10.0.0.13
  service:
    enabled: true
    port: 2381
    targetPort: 2381
  serviceMonitor:
    enabled: true
    scheme: http
    # Talos etcd does not require client certs for /metrics by default
    # If mTLS is enabled, configure the following:
    # caFile: /etc/prometheus/secrets/etcd-client-cert/ca.crt
    # certFile: /etc/prometheus/secrets/etcd-client-cert/tls.crt
    # keyFile: /etc/prometheus/secrets/etcd-client-cert/tls.key

# --- API Server ---
kubeApiServer:
  enabled: true

# --- kube-controller-manager ---
kubeControllerManager:
  enabled: true
  endpoints:
    - 10.0.0.11
    - 10.0.0.12
    - 10.0.0.13
  service:
    enabled: true
    port: 10257
    targetPort: 10257
  serviceMonitor:
    enabled: true
    https: true
    insecureSkipVerify: true

# --- kube-scheduler ---
kubeScheduler:
  enabled: true
  endpoints:
    - 10.0.0.11
    - 10.0.0.12
    - 10.0.0.13
  service:
    enabled: true
    port: 10259
    targetPort: 10259
  serviceMonitor:
    enabled: true
    https: true
    insecureSkipVerify: true

# --- kube-proxy ---
kubeProxy:
  enabled: false  # Talos with Cilium kube-proxy replacement does not run kube-proxy
```

Install the chart:

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --values prometheus-stack-values.yaml \
  --version 67.9.0

# Wait for all pods to be ready
kubectl -n monitoring wait pod --all --for=condition=ready --timeout=300s
```

## ServiceMonitor for Application Workloads

Create a ServiceMonitor to scrape any application that exposes Prometheus metrics:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
  labels:
    release: kube-prometheus-stack   # Must match the Helm release label selector
spec:
  namespaceSelector:
    matchNames:
      - my-app-namespace
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

## PrometheusRule — Recommended Alert Rules

The chart ships with hundreds of default rules. Add custom rules for Talos-specific concerns:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: talos-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: talos.rules
      rules:
        # etcd cluster health
        - alert: EtcdMembersDown
          expr: count(etcd_server_has_leader == 1) < 3
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "etcd cluster has fewer than 3 members with a leader"
            description: "Only {{ $value }} etcd members report having a leader. Investigate with talosctl etcd members."

        # etcd disk latency
        - alert: EtcdHighFsyncDuration
          expr: histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.5
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "etcd WAL fsync latency is high"
            description: "99th percentile etcd WAL fsync duration is {{ $value }}s on {{ $labels.instance }}."

        # etcd DB size
        - alert: EtcdDatabaseSizeGrowing
          expr: etcd_mvcc_db_total_size_in_bytes > 6442450944
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "etcd database size exceeds 6GB"
            description: "etcd on {{ $labels.instance }} has a database size of {{ $value | humanize1024 }}."

        # Node not ready
        - alert: KubeNodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.node }} is not Ready"
            description: "Node {{ $labels.node }} has been in NotReady state for more than 5 minutes."

        # High node CPU
        - alert: NodeHighCPU
          expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Node {{ $labels.instance }} CPU usage above 90%"
            description: "Node {{ $labels.instance }} has sustained CPU usage of {{ $value }}%."

        # Node memory pressure
        - alert: NodeHighMemory
          expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Node {{ $labels.instance }} memory usage above 90%"
            description: "Node {{ $labels.instance }} memory usage is {{ $value }}%."

        # PVC almost full
        - alert: PVCAlmostFull
          expr: kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} in {{ $labels.namespace }} is over 85% full"
            description: "PVC {{ $labels.persistentvolumeclaim }} is using {{ $value | humanizePercentage }} of capacity."

        # Prometheus TSDB compactions failing
        - alert: PrometheusTSDBCompactionsFailing
          expr: increase(prometheus_tsdb_compactions_failed_total[1h]) > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Prometheus TSDB compactions are failing"
            description: "Prometheus instance {{ $labels.instance }} has failing TSDB compactions."
```

Apply the rule:

```bash
kubectl apply -f talos-alerts.yaml
```

## Grafana Dashboards

The chart ships with dashboards for Kubernetes, node-exporter, etcd, and CoreDNS. Access Grafana:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
# Open http://localhost:3000
# Default credentials: admin / changeme-use-secret-in-production
```

### Import Additional Dashboards

Import community dashboards by ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-talos
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  talos-overview.json: |
    {
      "dashboard": {
        "title": "Talos Cluster Overview",
        "uid": "talos-overview",
        "panels": []
      }
    }
```

Popular Grafana dashboard IDs to import via the UI:

| Dashboard | ID | Description |
|---|---|---|
| Node Exporter Full | 1860 | Comprehensive node metrics |
| Kubernetes Cluster | 7249 | Cluster-wide overview |
| etcd | 3070 | etcd performance and health |
| CoreDNS | 15762 | DNS query metrics |

## Persistent Volume for Retention

Prometheus retention is controlled by two parameters:

- `retention: 15d` — time-based retention
- `retentionSize: "45GB"` — size-based retention (whichever triggers first)

The PV is configured in the values file under `prometheus.prometheusSpec.storageSpec`. Use a StorageClass that supports volume expansion so you can grow the PV without data loss:

```bash
# Check current disk usage
kubectl -n monitoring exec prometheus-kube-prometheus-stack-prometheus-0 -- \
  df -h /prometheus
```

To resize:

```bash
kubectl -n monitoring patch pvc prometheus-kube-prometheus-stack-prometheus-db-prometheus-kube-prometheus-stack-prometheus-0 \
  -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

## Verification

```bash
# Check all monitoring pods are running
kubectl -n monitoring get pods

# Verify Prometheus targets
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
# Open http://localhost:9090/targets — all targets should show UP

# Verify AlertManager
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093:9093
# Open http://localhost:9093 — check alert routing

# Test a PromQL query
curl -s 'http://localhost:9090/api/v1/query?query=up' | jq '.data.result | length'
```

## Upgrading

```bash
helm repo update
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-stack-values.yaml \
  --version 67.9.0
```

CRD updates must be applied manually before upgrading in some cases:

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/charts/crds/crds/crd-servicemonitors.yaml

kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/charts/crds/crds/crd-prometheusrules.yaml
```

## Operational Notes

- **Grafana admin password**: store it in a Kubernetes Secret and reference it with `grafana.admin.existingSecret` in production.
- **AlertManager secrets**: store Slack webhook URLs and PagerDuty keys in a Secret, then reference via `alertmanager.config.global.slack_api_url_file` or environment variables.
- **High availability**: set `prometheus.prometheusSpec.replicas: 2` and `alertmanager.alertmanagerSpec.replicas: 3` for HA. Use Thanos sidecar for cross-replica deduplication.
- **kube-proxy disabled**: if using Cilium as kube-proxy replacement, set `kubeProxy.enabled: false` in the values file (shown above).
