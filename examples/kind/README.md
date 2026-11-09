# Example: obsidian on kind

Install the stack on a local [kind](https://kind.sigs.k8s.io/) cluster:

```bash
kind create cluster --name obsidian

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana               https://grafana.github.io/helm-charts
helm repo add open-telemetry        https://open-telemetry.github.io/opentelemetry-helm-charts
helm dependency update charts/obsidian

helm upgrade --install obsidian charts/obsidian \
  --namespace observability --create-namespace

# Grafana
kubectl -n observability port-forward svc/obsidian-grafana 3000:80
# user: admin, password: changeme

# OTLP endpoints inside the cluster
# HTTP:  http://obsidian-opentelemetry-collector:4318
# gRPC:  obsidian-opentelemetry-collector:4317
```

Point any service instrumented with OpenTelemetry at the collector's OTLP endpoint and the telemetry will be routed to Tempo (traces), Prometheus remote-write (metrics) and Loki (logs).
