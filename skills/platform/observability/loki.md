# Loki on Talos

Loki is a log aggregation system that indexes only labels, not the full log text. Paired with Grafana Alloy (or Promtail) for collection, it provides centralized logging for Talos clusters at a fraction of the resource cost of Elasticsearch.

## Architecture

- **Loki** — stores and queries logs. Deployed in simple-scalable mode (read + write + backend components).
- **Grafana Alloy** — DaemonSet that tails container logs from `/var/log/pods` and ships them to Loki. Replaces the older Promtail agent.
- **Grafana** — query interface using LogQL. Already deployed by kube-prometheus-stack.

## Prerequisites

- A running Talos cluster with kubectl and Helm configured
- A default StorageClass for persistent volumes
- kube-prometheus-stack deployed (provides Grafana)

## Install Loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Create the values file:

```yaml
# loki-values.yaml

deploymentMode: SimpleScalable

loki:
  auth_enabled: false

  storage:
    type: filesystem

  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: index_
          period: 24h

  limits_config:
    retention_period: 168h              # 7 days
    max_query_length: 721h
    max_query_series: 5000
    ingestion_rate_mb: 10
    ingestion_burst_size_mb: 20
    per_stream_rate_limit: 3MB
    per_stream_rate_limit_burst: 15MB

  compactor:
    retention_enabled: true
    delete_request_store: filesystem
    compaction_interval: 10m
    retention_delete_delay: 2h
    retention_delete_worker_count: 150

  commonConfig:
    replication_factor: 1

# --- Write path ---
write:
  replicas: 2
  persistence:
    size: 10Gi
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      memory: 1Gi

# --- Read path ---
read:
  replicas: 2
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      memory: 1Gi

# --- Backend (compactor, ruler, etc.) ---
backend:
  replicas: 1
  persistence:
    size: 10Gi
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      memory: 512Mi

# --- Gateway ---
gateway:
  replicas: 1
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      memory: 128Mi

# Disable components not needed in simple-scalable mode
singleBinary:
  replicas: 0

# Monitoring
monitoring:
  serviceMonitor:
    enabled: true
    labels:
      release: kube-prometheus-stack
  rules:
    enabled: true
    labels:
      release: kube-prometheus-stack
```

Install:

```bash
helm install loki grafana/loki \
  --namespace loki --create-namespace \
  --values loki-values.yaml \
  --version 6.24.0

kubectl -n loki wait pod --all --for=condition=ready --timeout=300s
```

## Install Grafana Alloy (Log Collector)

Alloy replaces Promtail as the recommended log collector. It runs as a DaemonSet and tails container logs.

```yaml
# alloy-values.yaml

alloy:
  configMap:
    create: true
    content: |
      // Discover Kubernetes pods
      discovery.kubernetes "pods" {
        role = "pod"
      }

      // Relabel to extract metadata
      discovery.relabel "pods" {
        targets = discovery.kubernetes.pods.targets

        rule {
          source_labels = ["__meta_kubernetes_pod_node_name"]
          target_label  = "node"
        }
        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label  = "namespace"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          target_label  = "pod"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label  = "container"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
          target_label  = "app"
        }
      }

      // Tail container log files
      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pods.output
        forward_to = [loki.process.pipeline.receiver]
      }

      // Processing pipeline — parse JSON logs if present
      loki.process "pipeline" {
        stage.cri {}

        stage.json {
          expressions = {
            level  = "level",
            msg    = "msg",
            caller = "caller",
          }
          drop_malformed = true
        }

        stage.labels {
          values = {
            level = "",
          }
        }

        forward_to = [loki.write.default.receiver]
      }

      // Write to Loki
      loki.write "default" {
        endpoint {
          url = "http://loki-gateway.loki:80/loki/api/v1/push"
        }
      }

controller:
  type: daemonset

  tolerations:
    - operator: Exists

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      memory: 256Mi

  mounts:
    varlog: true     # Mount /var/log from host for container log access

serviceMonitor:
  enabled: true
  labels:
    release: kube-prometheus-stack
```

Install Alloy:

```bash
helm install alloy grafana/alloy \
  --namespace loki --create-namespace \
  --values alloy-values.yaml \
  --version 0.12.0

kubectl -n loki wait pod --all --for=condition=ready --timeout=120s
```

## Alternative: Promtail

If you prefer Promtail over Alloy:

```yaml
# promtail-values.yaml
config:
  clients:
    - url: http://loki-gateway.loki:80/loki/api/v1/push

tolerations:
  - operator: Exists

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi

serviceMonitor:
  enabled: true
  labels:
    release: kube-prometheus-stack
```

```bash
helm install promtail grafana/promtail \
  --namespace loki \
  --values promtail-values.yaml \
  --version 6.16.6
```

## Talos Service Log Collection

Container logs are handled by Alloy. For Talos-level service logs (machined, etcd, apid), configure the Talos machine config log sink:

```yaml
# Talos machine config patch
machine:
  logging:
    destinations:
      - endpoint: "tcp://10.0.0.50:5140"
        format: json_lines
```

Then deploy an Alloy instance (or a separate receiver) that listens on port 5140 and forwards to Loki:

```yaml
# alloy-talos-receiver config snippet
loki.source.syslog "talos" {
  listener {
    address  = "0.0.0.0:5140"
    protocol = "tcp"
    labels   = {
      source = "talos",
    }
  }
  forward_to = [loki.write.default.receiver]
}
```

This gives you Talos service logs alongside container logs in Grafana.

## Grafana Datasource

If you installed kube-prometheus-stack with the Loki datasource in the values file (see prometheus.md), Grafana already has Loki configured. Otherwise, add it manually:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
# Navigate to Configuration > Data Sources > Add Loki
# URL: http://loki-gateway.loki:80
```

## LogQL Examples

Query logs in Grafana Explore or via the Loki API.

### Basic Queries

```logql
# All logs from a namespace
{namespace="my-app"}

# Logs from a specific pod
{namespace="my-app", pod="api-server-0"}

# Logs from a specific container
{namespace="my-app", container="nginx"}

# Logs matching a regex on the pod name
{namespace="my-app", pod=~"api-server-.*"}
```

### Filter Expressions

```logql
# Lines containing "error" (case-insensitive)
{namespace="my-app"} |~ "(?i)error"

# Lines NOT containing "healthcheck"
{namespace="my-app"} !~ "healthcheck"

# JSON-parsed logs filtering by level
{namespace="my-app"} | json | level="error"

# Logfmt-parsed logs
{namespace="kube-system"} | logfmt | err!=""
```

### Aggregation Queries

```logql
# Error rate per namespace over the last hour
sum by (namespace) (count_over_time({namespace=~".+"} |~ "(?i)error" [1h]))

# Log volume (bytes/sec) by pod
sum by (pod) (bytes_rate({namespace="my-app"} [5m]))

# Top 10 pods by log volume
topk(10, sum by (pod) (bytes_rate({namespace=~".+"} [5m])))

# Error rate as a percentage of total logs
sum(count_over_time({namespace="my-app"} |~ "(?i)error" [5m]))
/
sum(count_over_time({namespace="my-app"} [5m])) * 100
```

### Talos Service Logs

```logql
# All Talos service logs
{source="talos"}

# etcd logs from Talos
{source="talos"} |~ "etcd"

# Talos machined errors
{source="talos"} | json | talos_service="machined" | level="error"
```

## Retention Configuration

Retention is configured in the Loki values under `loki.limits_config.retention_period`. The compactor enforces retention:

| Setting | Value | Effect |
|---|---|---|
| `retention_period` | `168h` (7d) | Chunks older than 7 days are deleted |
| `compaction_interval` | `10m` | How often the compactor runs |
| `retention_delete_delay` | `2h` | Grace period before deletion |

To increase retention to 30 days:

```yaml
loki:
  limits_config:
    retention_period: 720h
```

Ensure the write and backend PVs have enough capacity. Estimate roughly 5-10 GB per day for a moderately active 5-node cluster.

## Verification

```bash
# Check all Loki pods are running
kubectl -n loki get pods

# Check Alloy is collecting logs
kubectl -n loki logs daemonset/alloy --tail=20

# Query Loki directly
kubectl -n loki port-forward svc/loki-gateway 3100:80
curl -s 'http://localhost:3100/loki/api/v1/labels' | jq .

# Check log volume
curl -s 'http://localhost:3100/loki/api/v1/query?query=sum(bytes_rate({namespace=~".%2B"}[5m]))' | jq .
```

## Operational Notes

- **Multi-tenancy**: set `loki.auth_enabled: true` and configure `X-Scope-OrgID` headers in Alloy and Grafana for tenant isolation.
- **Object storage**: for production at scale, switch from `filesystem` to S3-compatible storage (MinIO, AWS S3) by changing `loki.storage.type` to `s3` and providing bucket configuration.
- **High availability**: the simple-scalable mode with 2 write replicas and 2 read replicas provides HA for most clusters. For large-scale deployments, use the microservices mode.
- **Log pipeline tuning**: drop noisy logs in the Alloy pipeline to reduce storage cost. Add `stage.drop` rules for health check endpoints, debug-level logs, or high-volume namespaces.
