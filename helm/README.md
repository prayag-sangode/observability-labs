#  Kubernetes Observability Stack - Step-by-Step Helm Setup with Custom Values

This guide sets up a complete observability stack using **Helm** with individual `values.yaml` files for:
- Prometheus (metrics)
- Grafana (dashboards)
- Loki & Promtail (logs)
- Jaeger (traces)
- OpenTelemetry Collector (tracing ingest)
- Node Exporter (host-level metrics)

---

##  Step 1: Create Folder for YAML Files

```bash
mkdir -p observability-values
````

---

##  Step 2: Generate `values.yaml` Files

###  Prometheus

Scrapes metrics from Node Exporter, OTEL Collector, and itself. PVC is enabled.

```bash
cat <<EOF > observability-values/prometheus-values.yaml
alertmanager:
  enabled: false

server:
  persistentVolume:
    enabled: true
    size: 8Gi

  service:
    type: ClusterIP

  extraScrapeConfigs: |
    - job_name: 'node-exporter'
      static_configs:
        - targets: ['node-exporter.monitoring.svc.cluster.local:9100']
    - job_name: 'otel-collector'
      static_configs:
        - targets: ['otel-collector.observability.svc.cluster.local:8888']
    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']
EOF
```

---

###  Grafana

Includes data sources for Prometheus, Loki, and Jaeger. PVC and NodePort enabled.

```bash
cat <<EOF > observability-values/grafana-values.yaml
adminPassword: admin

persistence:
  enabled: true
  size: 5Gi

service:
  type: NodePort

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://prometheus-server.monitoring.svc.cluster.local
      - name: Loki
        type: loki
        access: proxy
        url: http://loki.observability.svc.cluster.local:3100
      - name: Jaeger
        type: jaeger
        access: proxy
        url: http://jaeger-query.observability.svc.cluster.local
EOF
```

---

###  Loki

Stores logs collected by Promtail. PVC enabled.

```bash
cat <<EOF > observability-values/loki-values.yaml
persistence:
  enabled: true
  accessModes:
    - ReadWriteOnce
  size: 10Gi
EOF
```

---

###  Promtail

Log collector that pushes logs to Loki.

```bash
cat <<EOF > observability-values/promtail-values.yaml
loki:
  serviceName: loki

config:
  clients:
    - url: http://loki.observability.svc.cluster.local:3100/loki/api/v1/push

  snippets:
    pipelineStages:
      - docker: {}
      - cri: {}
EOF
```

---

###  Jaeger

Used to visualize distributed traces. Using in-memory storage and NodePort access.

```bash
cat <<EOF > observability-values/jaeger-values.yaml
storage:
  type: memory

provisionDataStore:
  cassandra: false

query:
  service:
    type: NodePort

collector:
  service:
    type: ClusterIP
EOF
```

---

###  OpenTelemetry Collector

Collects traces from your apps and sends to Jaeger. Also logs them to console.

```bash
cat <<EOF > observability-values/otel-collector-values.yaml
mode: deployment

config:
  receivers:
    otlp:
      protocols:
        grpc:
        http:

  exporters:
    logging:
      loglevel: debug
    otlp:
      endpoint: jaeger-collector.observability.svc.cluster.local:4317

  processors:
    batch: {}

  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [batch]
        exporters: [logging, otlp]
EOF
```

---

### Node Exporter

Exports node-level metrics to Prometheus.

```bash
cat <<EOF > observability-values/node-exporter-values.yaml
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
EOF
```

---

##  Step 3: Add Helm Repos

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

---

##  Step 4: Install All Components

```bash
kubectl create namespace monitoring
kubectl create namespace observability

helm upgrade --install prometheus prometheus-community/prometheus \
  -n monitoring -f observability-values/prometheus-values.yaml

helm upgrade --install grafana/grafana \
  -n monitoring -f observability-values/grafana-values.yaml

helm upgrade --install node-exporter prometheus-community/prometheus-node-exporter \
  -n monitoring -f observability-values/node-exporter-values.yaml

helm upgrade --install loki grafana/loki \
  -n observability -f observability-values/loki-values.yaml

helm upgrade --install promtail grafana/promtail \
  -n observability -f observability-values/promtail-values.yaml

helm upgrade --install jaeger jaegertracing/jaeger \
  -n observability -f observability-values/jaeger-values.yaml

helm upgrade --install otel-collector open-telemetry/opentelemetry-collector \
  -n observability -f observability-values/otel-collector-values.yaml
```

---

##  Step 5: Access Dashboards & UIs

| Component      | Access Instructions                                                                                       |
| -------------- | --------------------------------------------------------------------------------------------------------- |
| **Grafana**    | `kubectl port-forward svc/grafana 3000:80 -n monitoring` → [http://localhost:3000](http://localhost:3000) |
| **Prometheus** | `kubectl port-forward svc/prometheus-server 9090:80 -n monitoring`                                        |
| **Jaeger**     | `kubectl port-forward svc/jaeger-query 16686:16686 -n observability`                                      |

Default Grafana login:

* Username: `admin`
* Password: `admin`

---

##  Optional Cleanup

```bash
helm uninstall prometheus grafana node-exporter -n monitoring
helm uninstall loki promtail jaeger otel-collector -n observability
kubectl delete ns monitoring observability
```

---

##  Summary

This setup gives you:

* **Metrics** via Prometheus & Node Exporter
* **Logs** via Promtail → Loki → Grafana
* **Traces** via OTEL Collector → Jaeger → Grafana

It uses persistent volumes (PVCs) and can be extended to production easily.

