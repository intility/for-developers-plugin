# Manifest Templates

YAML templates for Collector and PodLogCollector. Substitute placeholders, write to disk, apply.

## Placeholders

### Collector

| Placeholder | Example |
|---|---|
| `<name>` | `platform-collector` |
| `<namespace>` | `otel-operator-system` |
| `<otlp-endpoint>` | `tempo.observability.svc.cluster.local:4317` |
| `<secret-name>` | `tempo-write-token` |
| `<secret-key>` | `token` |

### PodLogCollector

| Placeholder | Example |
|---|---|
| `<name>` | `platform-podlogcollector` |
| `<namespace>` | `otel-operator-system` |
| `<log-endpoint>` | `loki.observability.svc.cluster.local:4317` |
| `<selector-label-key>` | `collect-podlogs` |
| `<selector-label-value>` | `"true"` |

---

## Collector

Runs as a Deployment. Receives OTLP traces and metrics, exports to a backend.

```yaml
apiVersion: otel.intility.io/v1alpha1
kind: Collector
metadata:
  name: <name>
  namespace: <namespace>
  labels:
    app.kubernetes.io/name: otel-operator
spec:
  replicas: 1
  allowProcessorOverride: true

  service:
    type: ClusterIP
    ports:
      - name: otlp-grpc
        port: 4317
        targetPort: 4317
        protocol: TCP
      - name: otlp-http
        port: 4318
        targetPort: 4318
        protocol: TCP

  config: |
    extensions:
      health_check:
        endpoint: 0.0.0.0:13133

    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_mib: 400

    exporters:
      # Use this block for gRPC backends (port 4317):
      otlp:
        endpoint: <otlp-endpoint>          # e.g. tempo.observability.svc.cluster.local:4317
        tls:
          insecure: true
      # OR use this block for HTTP backends (port 4318 or Loki on 3100).
      # otlphttp appends /v1/traces and /v1/metrics automatically — provide base URL only, no path suffix.
      # otlphttp:
      #   endpoint: http://<svc>.<ns>.svc.cluster.local:<port>

    service:
      extensions: [health_check]
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp]
```

### With authentication (external endpoints)

Add `secrets` to the spec and reference the token as a file in the exporter. The operator mounts each key in the secret as a file at `<mountPath>/<key>`.

```yaml
spec:
  secrets:
    - name: <secret-name>
      mountPath: /secrets/backend
  config: |
    exporters:
      otlp:
        endpoint: <otlp-endpoint>
        headers:
          authorization: "Bearer ${file:/secrets/backend/<secret-key>}"
        tls:
          insecure: false
```

Never put the token value directly in the manifest — always reference it via `${file:...}`.

### Traces only

Remove the `metrics` pipeline from the `service.pipelines` block.

### Metrics only via Prometheus scrape endpoint

Replace the exporter and pipeline:

```yaml
    exporters:
      prometheus:
        endpoint: "0.0.0.0:8889"

    service:
      extensions: [health_check]
      pipelines:
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
```

Add a `metrics` port to the service spec:

```yaml
      - name: metrics
        port: 8889
        targetPort: 8889
        protocol: TCP
```

---

## PodLogCollector

Runs as a DaemonSet. Tails `/var/log/pods/` on every node and ships logs from labelled pods.

The operator auto-creates the ServiceAccount, RBAC, and privileged SCC binding — don't add `serviceAccountName` unless you need a custom SA.

The `podSelector` field is optional: when omitted, the operator defaults to `collect-podlogs: "true"`. Only include it when using a custom label.

```yaml
apiVersion: otel.intility.io/v1alpha1
kind: PodLogCollector
metadata:
  name: <name>
  namespace: <namespace>
  labels:
    app.kubernetes.io/name: otel-operator
spec:
  allowProcessorOverride: true

  # Only needed when using a non-default label selector.
  # Remove these lines to use the default: collect-podlogs: "true"
  podSelector:
    matchLabels:
      <selector-label-key>: <selector-label-value>

  config: |
    receivers:
      filelog:
        include:
          - /var/log/pods/*/*/*.log
        exclude:
          - /var/log/pods/*/otel-collector/*.log
        start_at: end
        include_file_path: true
        include_file_name: false
        operators:
          - id: container-parser
            type: container

    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_mib: 400

    exporters:
      # Use for gRPC backends (port 4317):
      otlp:
        endpoint: <log-endpoint>           # e.g. loki.monitoring.svc.cluster.local:4317
        tls:
          insecure: true
      # OR use for HTTP backends (Loki on port 3100, or any port 4318).
      # otlphttp appends /v1/logs automatically — provide base URL only, no path suffix.
      # otlphttp:
      #   endpoint: http://<svc>.<ns>.svc.cluster.local:<port>

    service:
      pipelines:
        logs:
          receivers: [filelog]
          processors: [memory_limiter, batch]
          exporters: [otlp]   # change to [otlphttp] if using the HTTP exporter
```

### With authentication (external endpoints)

Same pattern as Collector — add `secrets` to the spec and reference the token via `${file:...}`:

```yaml
spec:
  secrets:
    - name: <secret-name>
      mountPath: /secrets/backend
  config: |
    exporters:
      otlp:
        endpoint: <log-endpoint>
        headers:
          authorization: "Bearer ${file:/secrets/backend/<secret-key>}"
        tls:
          insecure: false
```

### Debug mode (testing only)

If the user wants to verify log collection without a real backend, replace the exporter:

```yaml
    exporters:
      debug: {}

    service:
      pipelines:
        logs:
          receivers: [filelog]
          processors: [memory_limiter, batch]
          exporters: [debug]
```

Logs appear in:
```bash
oc logs -n <namespace> -l app.kubernetes.io/component=otel-podlog-collector
```

---

## Full minimal examples

### Collector — traces and metrics to Tempo

```yaml
apiVersion: otel.intility.io/v1alpha1
kind: Collector
metadata:
  name: platform-collector
  namespace: otel-operator-system
  labels:
    app.kubernetes.io/name: otel-operator
spec:
  replicas: 1
  allowProcessorOverride: true
  service:
    type: ClusterIP
    ports:
      - name: otlp-grpc
        port: 4317
        targetPort: 4317
        protocol: TCP
      - name: otlp-http
        port: 4318
        targetPort: 4318
        protocol: TCP
  config: |
    extensions:
      health_check:
        endpoint: 0.0.0.0:13133
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_mib: 400
    exporters:
      otlp:
        endpoint: tempo.observability.svc.cluster.local:4317
        tls:
          insecure: true
    service:
      extensions: [health_check]
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp]
```

### PodLogCollector — logs to Loki (default label)

```yaml
apiVersion: otel.intility.io/v1alpha1
kind: PodLogCollector
metadata:
  name: platform-podlogcollector
  namespace: otel-operator-system
  labels:
    app.kubernetes.io/name: otel-operator
spec:
  allowProcessorOverride: true
  config: |
    receivers:
      filelog:
        include:
          - /var/log/pods/*/*/*.log
        exclude:
          - /var/log/pods/*/otel-collector/*.log
        start_at: end
        include_file_path: true
        include_file_name: false
        operators:
          - id: container-parser
            type: container
    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_mib: 400
    exporters:
      otlp:
        endpoint: loki.observability.svc.cluster.local:4317
        tls:
          insecure: true
    service:
      pipelines:
        logs:
          receivers: [filelog]
          processors: [memory_limiter, batch]
          exporters: [otlp]
```
