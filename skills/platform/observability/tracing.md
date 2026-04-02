# Tracing on Talos

Distributed tracing with OpenTelemetry Collector and a Grafana-integrated backend (Tempo or Jaeger). The OTEL Collector acts as a vendor-neutral pipeline that receives traces from applications, processes them, and exports to the backend.

## Architecture

```
Application (OTLP) --> OTEL Collector --> Tempo/Jaeger --> Grafana
```

- **Applications** instrument with OpenTelemetry SDKs or auto-instrumentation and send traces via OTLP (gRPC or HTTP).
- **OTEL Collector** receives, batches, samples, and exports traces. Deployed as a Deployment (gateway mode) or DaemonSet (agent mode).
- **Tempo** stores traces with minimal indexing, queried through Grafana. Alternative: Jaeger with Elasticsearch or Cassandra backend.
- **Grafana** provides the trace visualization UI, linked to Loki logs and Prometheus metrics.

## Prerequisites

- A running Talos cluster with kubectl and Helm configured
- kube-prometheus-stack deployed (provides Grafana)
- A StorageClass for Tempo PVCs (optional for Jaeger in-memory mode)

## Install Tempo (Recommended Backend)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

```yaml
# tempo-values.yaml

tempo:
  storage:
    trace:
      backend: local
      local:
        path: /var/tempo/traces
      wal:
        path: /var/tempo/wal

  retention: 168h  # 7 days

  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: "0.0.0.0:4317"
        http:
          endpoint: "0.0.0.0:4318"
    zipkin:
      endpoint: "0.0.0.0:9411"
    jaeger:
      protocols:
        thrift_http:
          endpoint: "0.0.0.0:14268"
        grpc:
          endpoint: "0.0.0.0:14250"

  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      memory: 1Gi

persistence:
  enabled: true
  size: 10Gi

serviceMonitor:
  enabled: true
  labels:
    release: kube-prometheus-stack

tempoQuery:
  enabled: true   # Enables the Jaeger-compatible query frontend
```

Install:

```bash
helm install tempo grafana/tempo \
  --namespace tempo --create-namespace \
  --values tempo-values.yaml \
  --version 1.12.0

kubectl -n tempo wait pod --all --for=condition=ready --timeout=300s
```

## Alternative: Jaeger Backend

If you prefer Jaeger over Tempo:

```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update
```

```yaml
# jaeger-values.yaml

provisionDataStore:
  cassandra: false
  elasticsearch: false

allInOne:
  enabled: true
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      memory: 1Gi
  extraEnv:
    - name: SPAN_STORAGE_TYPE
      value: badger
    - name: BADGER_EPHEMERAL
      value: "false"
    - name: BADGER_DIRECTORY_VALUE
      value: /badger/data
    - name: BADGER_DIRECTORY_KEY
      value: /badger/key

storage:
  type: badger

  badger:
    persistence:
      enabled: true
      size: 10Gi
```

```bash
helm install jaeger jaegertracing/jaeger \
  --namespace jaeger --create-namespace \
  --values jaeger-values.yaml \
  --version 3.4.1

kubectl -n jaeger wait pod --all --for=condition=ready --timeout=300s
```

## Install OpenTelemetry Collector

The OTEL Collector sits between applications and the trace backend. Deploy in gateway mode (Deployment):

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

```yaml
# otel-collector-values.yaml

mode: deployment
replicaCount: 2

config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: "0.0.0.0:4317"
        http:
          endpoint: "0.0.0.0:4318"

  processors:
    batch:
      timeout: 5s
      send_batch_size: 1024
      send_batch_max_size: 2048

    memory_limiter:
      check_interval: 1s
      limit_mib: 512
      spike_limit_mib: 128

    # Tail-based sampling — keep errors and slow traces, sample the rest
    tail_sampling:
      decision_wait: 10s
      num_traces: 50000
      expected_new_traces_per_sec: 100
      policies:
        # Always keep traces with errors
        - name: errors
          type: status_code
          status_code:
            status_codes:
              - ERROR
        # Always keep traces slower than 2s
        - name: slow-traces
          type: latency
          latency:
            threshold_ms: 2000
        # Sample 10% of remaining traces
        - name: probabilistic
          type: probabilistic
          probabilistic:
            sampling_percentage: 10

  exporters:
    # Export to Tempo
    otlp/tempo:
      endpoint: "tempo.tempo:4317"
      tls:
        insecure: true

    # If using Jaeger instead:
    # otlp/jaeger:
    #   endpoint: "jaeger-collector.jaeger:4317"
    #   tls:
    #     insecure: true

    # Debug exporter for troubleshooting
    debug:
      verbosity: basic

  extensions:
    health_check:
      endpoint: "0.0.0.0:13133"

  service:
    extensions: [health_check]
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, tail_sampling, batch]
        exporters: [otlp/tempo]

ports:
  otlp:
    enabled: true
    containerPort: 4317
    servicePort: 4317
    protocol: TCP
  otlp-http:
    enabled: true
    containerPort: 4318
    servicePort: 4318
    protocol: TCP

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    memory: 512Mi

serviceMonitor:
  enabled: true
  labels:
    release: kube-prometheus-stack
```

Install:

```bash
helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace otel --create-namespace \
  --values otel-collector-values.yaml \
  --version 0.107.0

kubectl -n otel wait pod --all --for=condition=ready --timeout=120s
```

## Application Instrumentation

### Environment Variables for Auto-Instrumentation

Configure applications to send traces to the OTEL Collector by setting standard OTEL environment variables:

```yaml
# In your application Deployment spec
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector-opentelemetry-collector.otel:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  - name: OTEL_SERVICE_NAME
    value: "my-service"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment=production,service.namespace=my-app"
```

### OpenTelemetry Operator Auto-Instrumentation

The OTEL Operator can inject instrumentation into applications without code changes:

```bash
helm install otel-operator open-telemetry/opentelemetry-operator \
  --namespace otel \
  --set admissionWebhooks.certManager.enabled=true \
  --version 0.74.0
```

Create an Instrumentation resource:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: auto-instrumentation
  namespace: my-app
spec:
  exporter:
    endpoint: http://otel-collector-opentelemetry-collector.otel:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"    # Sample 25% at the head
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:2.12.0
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:0.57.0
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.52b0
  dotnet:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-dotnet:1.9.0
  go:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-go:0.18.0
```

Annotate pods for auto-instrumentation:

```yaml
# Java application
metadata:
  annotations:
    instrumentation.opentelemetry.io/inject-java: "my-app/auto-instrumentation"

# Node.js application
metadata:
  annotations:
    instrumentation.opentelemetry.io/inject-nodejs: "my-app/auto-instrumentation"

# Python application
metadata:
  annotations:
    instrumentation.opentelemetry.io/inject-python: "my-app/auto-instrumentation"

# .NET application
metadata:
  annotations:
    instrumentation.opentelemetry.io/inject-dotnet: "my-app/auto-instrumentation"

# Go application (requires eBPF, limited support)
metadata:
  annotations:
    instrumentation.opentelemetry.io/inject-go: "my-app/auto-instrumentation"
```

## Sampling Strategies

| Strategy | Where | Use Case |
|---|---|---|
| Head-based (probabilistic) | SDK or Instrumentation CR | Simple, low overhead, decides at trace start |
| Tail-based | OTEL Collector | Keeps errors and slow traces, samples the rest |
| Always-on | SDK | Development, low-traffic services |
| Rate limiting | OTEL Collector | Fixed traces-per-second cap |

### Head-Based Sampling (at the SDK level)

Set in the Instrumentation CR or via environment variable:

```yaml
env:
  - name: OTEL_TRACES_SAMPLER
    value: "parentbased_traceidratio"
  - name: OTEL_TRACES_SAMPLER_ARG
    value: "0.1"   # 10% sampling
```

### Tail-Based Sampling (at the Collector)

Configured in the OTEL Collector config (shown above). Tail-based sampling requires the Collector to buffer traces before making a decision, which increases memory usage. The `decision_wait` parameter controls how long to wait for a trace to complete.

### Combined Strategy

For production, combine both:
1. Head-based sampling at 25% in the SDK (reduces data volume before it hits the Collector)
2. Tail-based sampling in the Collector (ensures 100% of errors and slow traces are kept)

## Grafana Integration

### Add Tempo as a Datasource

If you included Tempo in the kube-prometheus-stack values (see prometheus.md), it is already configured. Otherwise:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
# Navigate to Configuration > Data Sources > Add Tempo
# URL: http://tempo-query-frontend.tempo:3100
```

### Trace-to-Logs Correlation

Configure the Tempo datasource in Grafana to link traces to Loki logs:

```yaml
# In the Grafana datasource configuration (or via kube-prometheus-stack values)
jsonData:
  tracesToLogsV2:
    datasourceUid: "<loki-datasource-uid>"
    filterByTraceID: true
    filterBySpanID: false
    customQuery: true
    query: '{namespace="$${__span.tags.namespace}"} | json | trace_id="$${__trace.traceId}"'
  tracesToMetrics:
    datasourceUid: "<prometheus-datasource-uid>"
    spanStartTimeShift: "-1h"
    spanEndTimeShift: "1h"
    queries:
      - name: "Request rate"
        query: "sum(rate(http_server_request_duration_seconds_count{service_name=\"$${__span.tags[\"service.name\"]}\"}[5m]))"
  nodeGraph:
    enabled: true
  serviceMap:
    datasourceUid: "<prometheus-datasource-uid>"
```

### Explore Traces in Grafana

```
1. Open Grafana > Explore
2. Select the Tempo datasource
3. Use TraceQL to query:
   - { status = error }                          -- all error traces
   - { span.http.status_code >= 500 }           -- server errors
   - { duration > 2s }                           -- slow traces
   - { resource.service.name = "api-server" }   -- traces from a specific service
   - { name = "POST /api/orders" && duration > 1s }  -- slow endpoint
```

## Verification

```bash
# Check OTEL Collector is running
kubectl -n otel get pods

# Check Tempo is running
kubectl -n tempo get pods

# Verify OTEL Collector health
kubectl -n otel port-forward svc/otel-collector-opentelemetry-collector 13133:13133
curl -s http://localhost:13133/  # Should return {"status":"Server available"}

# Send a test trace via OTLP HTTP
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{"key": "service.name", "value": {"stringValue": "test-service"}}]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "5B8EFFF798038103D269B633813FC60C",
          "spanId": "EEE19B7EC3C1B174",
          "name": "test-span",
          "kind": 1,
          "startTimeUnixNano": "1544712660000000000",
          "endTimeUnixNano": "1544712661000000000",
          "status": {"code": 1}
        }]
      }]
    }]
  }'

# Query Tempo for the test trace
kubectl -n tempo port-forward svc/tempo 3100:3100
curl -s "http://localhost:3100/api/traces/5B8EFFF798038103D269B633813FC60C" | jq .
```

## Operational Notes

- **Resource overhead**: the OTEL Collector with tail-based sampling buffers traces in memory. Size `memory_limiter` and `num_traces` based on expected throughput.
- **DaemonSet mode**: for high-traffic clusters, deploy the Collector as a DaemonSet (agent mode) on each node to reduce network hops, with a central gateway Deployment for sampling and export.
- **Tempo scaling**: for production at scale, switch Tempo to the distributed mode with S3/GCS backend. The monolithic mode shown here is suitable for clusters with up to ~50 spans/second sustained.
- **Trace retention**: Tempo's `retention` setting controls how long traces are kept. 7 days is a sensible default; adjust based on debugging needs and storage capacity.
- **cert-manager**: the OTEL Operator auto-instrumentation webhook requires cert-manager. Install it before the operator if not already present.
