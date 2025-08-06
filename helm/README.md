#  Kubernetes Observability Stack — Helm Chart Setup

This setup deploys a complete observability stack in Kubernetes using **independent Helm charts** for each component:

- Prometheus (metrics)
- Grafana (dashboards)
- Alertmanager (if enabled)
- Node Exporter (node-level metrics)
- Loki (log storage)
- Promtail (log collector)
- Jaeger (traces)
- OpenTelemetry Collector (metrics, traces)

---

##  Folder Structure

```

observability-values/
├── prometheus-values.yaml
├── grafana-values.yaml
├── loki-values.yaml
├── promtail-values.yaml
├── jaeger-values.yaml
├── otel-collector-values.yaml
├── node-exporter-values.yaml
└── README.md

````

---

##  Helm Repositories Setup

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
````

---

##  Installation Commands

```bash
kubectl create namespace monitoring
kubectl create namespace observability

# Monitoring Stack
helm install prometheus prometheus-community/prometheus \
  -n monitoring -f prometheus-values.yaml

helm install grafana grafana/grafana \
  -n monitoring -f grafana-values.yaml

helm install node-exporter prometheus-community/prometheus-node-exporter \
  -n monitoring -f node-exporter-values.yaml

# Observability Stack
helm install loki grafana/loki \
  -n observability -f loki-values.yaml

helm install promtail grafana/promtail \
  -n observability -f promtail-values.yaml

helm install jaeger jaegertracing/jaeger \
  -n observability -f jaeger-values.yaml

helm install otel-collector open-telemetry/opentelemetry-collector \
  -n observability -f otel-collector-values.yaml
```

---

##  Component Integration

| Component      | Purpose                      | Sends Data To            |
| -------------- | ---------------------------- | ------------------------ |
| Prometheus     | Metrics database             | Grafana                  |
| Node Exporter  | Node-level metrics           | Prometheus               |
| Grafana        | Dashboards/Visualization     | Prometheus, Loki, Jaeger |
| Loki           | Log aggregation backend      | Grafana                  |
| Promtail       | Log collection from nodes    | Loki                     |
| Jaeger         | Trace storage and query      | Grafana                  |
| OTEL Collector | Collects metrics/traces/logs | Jaeger + Loki            |

---

##  PVC Configurations

* Prometheus: 8Gi PVC
* Grafana: 5Gi PVC
* Loki: 10Gi PVC

These are persistent across pod restarts and store historical metrics and logs.

---

##  Credentials

* Grafana Admin: `admin / admin` (set in `grafana-values.yaml`)
* Update these as needed in a secure environment.

---

##  Accessing UIs

| Component  | Command or URL                                                       |
| ---------- | -------------------------------------------------------------------- |
| Grafana    | `kubectl port-forward svc/grafana 3000:80 -n monitoring`             |
| Prometheus | `kubectl port-forward svc/prometheus-server 9090:80 -n monitoring`   |
| Jaeger     | `kubectl port-forward svc/jaeger-query 16686:16686 -n observability` |

---

##  Testing

* Verify Prometheus metrics at: `http://localhost:9090`
* Log into Grafana: `http://localhost:3000` and import dashboards
* Check logs in Loki via Grafana > Explore
* View traces in Grafana > Tempo/Jaeger tab

---

##  Cleanup

```bash
helm uninstall prometheus grafana node-exporter -n monitoring
helm uninstall loki promtail jaeger otel-collector -n observability

kubectl delete ns monitoring observability
```

