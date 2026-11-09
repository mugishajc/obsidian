# obsidian

An opinionated observability stack for small Kubernetes clusters, wrapped as one Helm chart. Prometheus for metrics, Loki for logs, Tempo for traces, OpenTelemetry Collector as the one OTLP endpoint for everything. Ship a service, point its SDK at the collector, and all three signals arrive in Grafana.

## What is in the chart

| Signal | Component | Notes |
|---|---|---|
| Metrics | kube-prometheus-stack | Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics |
| Logs | Loki (single binary) | filesystem storage, TSDB schema v13 |
| Traces | Tempo | OTLP gRPC on 4317, OTLP HTTP on 4318, 72h retention |
| Ingest | OpenTelemetry Collector | one OTLP endpoint for metrics, logs and traces, with memory_limiter + batch processors |

Each sub-chart is behind a boolean flag so you can grow the stack piece by piece: metrics first, then add logs, then traces.

## Install

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana               https://grafana.github.io/helm-charts
helm repo add open-telemetry        https://open-telemetry.github.io/opentelemetry-helm-charts
helm dependency update charts/obsidian

helm upgrade --install obsidian charts/obsidian \
  --namespace observability --create-namespace
```

See `examples/kind/README.md` for a local walkthrough on a kind cluster.

## What ships for free

**Dashboards** (as `ConfigMap`s picked up by the Grafana sidecar):

- `dashboards/slo-overview.json`: 30-day availability, 28-day error-budget burn, p95 latency per route, RED metrics per service.
- `dashboards/logs-triage.json`: log volume by level, recent errors with labels.

**Alerts** (`PrometheusRule`):

- `alerts/slo-burn.yaml`: multi-window multi-burn-rate SLO burn alerts (14.4x in 1h is critical, 6x in 6h is warning), targeting a 99.9% SLO.

## OTLP endpoints inside the cluster

```
HTTP:  http://obsidian-opentelemetry-collector:4318
gRPC:  obsidian-opentelemetry-collector:4317
```

Any OpenTelemetry SDK can send to one endpoint; the collector routes traces to Tempo, metrics to Prometheus (via remote-write) and logs to Loki.

## Design

- **One collector, three backends.** Services do not know which store holds their telemetry. Swapping Tempo for Jaeger is a chart value change.
- **Single-binary Loki.** Enough for small clusters, keeps the operational cost near zero. Swap to the distributed mode by changing one value.
- **Dashboards in the chart.** No manual `grafana-import` step; a new JSON file under `dashboards/` is enough.
- **Alerts as Prometheus rules, not Alertmanager configs.** So the rules live beside the dashboards they explain.

## CI

- `helm lint` and `helm template` the chart (with dependencies fetched) on every PR
- every file under `dashboards/` is parsed as JSON
- every file under `alerts/` is parsed as YAML

## Not included

- Long-term metric storage. For retention beyond 15 days, point Prometheus remote-write at Mimir or Cortex.
- Blackbox probes. Add a `probes/` folder with `Probe` resources when needed.
- RBAC and ingress. Both depend on your cluster, so they are intentionally out of scope here.
