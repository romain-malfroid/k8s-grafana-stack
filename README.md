# k8s-grafana-stack

Helm umbrella chart for a complete Kubernetes observability stack: metrics, logs, traces and collection via Alloy.

## Stack

| Component | Chart | Version | Role |
|-----------|-------|---------|------|
| **Alloy** (via k8s-monitoring) | `grafana/k8s-monitoring` | 3.8.4 | Collector — scrapes everything, replaces node-exporter + kube-state-metrics + Promtail |
| **Prometheus** | `prometheus-community/prometheus` | 28.13.0 | Metrics storage — receives via remote-write from Alloy |
| **Alertmanager** | `prometheus-community/alertmanager` | 1.33.1 | Alerting |
| **Loki** | `grafana/loki` | 6.55.0 | Log storage (SingleBinary + MinIO) |
| **Tempo** | `grafana/tempo` | 1.24.4 | Trace storage (OTLP gRPC/HTTP) |
| **Grafana** | `grafana/grafana` | 10.5.15 | Visualization — pre-configured datasources |

**Architecture:**
```
Alloy (scrapes cluster) ──remote-write──► Prometheus (storage)
Alloy (pod logs)        ──push──────────► Loki (storage)
Apps (OTLP traces)      ──push──────────► Tempo (storage)
                                           ↑
                                        Grafana (queries all three)
```

> **Simpler alternative:** If you only need metrics + Grafana (no logs, no traces), use [`kube-prometheus-stack`](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) from prometheus-community. It bundles Prometheus Operator, Grafana, node-exporter and kube-state-metrics in a single chart.
>
> This chart is recommended when you need the full MELT stack (Metrics, Events, Logs, Traces) and want Grafana Labs' maintained collection layer (Alloy / k8s-monitoring).

---

## Install

### Any Kubernetes distribution

```bash
helm repo add k8s-grafana-stack https://romain-malfroid.github.io/k8s-grafana-stack
helm repo update
helm install k8s-grafana-stack k8s-grafana-stack/k8s-grafana-stack -n monitoring --create-namespace
```

### k3s (Traefik Gateway API)

Download the k3s values template, fill in your gateway name and domain, then install:

```bash
helm repo add k8s-grafana-stack https://romain-malfroid.github.io/k8s-grafana-stack
helm repo update

curl -O https://raw.githubusercontent.com/romain-malfroid/k8s-grafana-stack/main/charts/k8s-grafana-stack/values-k3s.yaml
# Edit values-k3s.yaml: set gatewayName, gatewayNamespace, root_url

helm install k8s-grafana-stack k8s-grafana-stack/k8s-grafana-stack \
  -f values-k3s.yaml -n monitoring --create-namespace
```

### Upgrade

```bash
helm upgrade k8s-grafana-stack k8s-grafana-stack/k8s-grafana-stack \
  [-f values-k3s.yaml] -n monitoring
```

---

## Configuration

`values-k3s.yaml` is the only file you need to customize. Key settings:

| Key | Description |
|-----|-------------|
| `traefik.gatewayName` | Your Traefik Gateway resource name |
| `traefik.gatewayNamespace` | Namespace where the Gateway lives |
| `grafana.grafana.ini.server.root_url` | Your domain + `/grafana` |
| `k8s-monitoring.cluster.name` | Cluster name shown in dashboards |
| `prometheus.server.retention` | Metrics retention (default: `7d`) |
| `grafana.adminPassword` | Grafana admin password (default: `admin`) |

---

## Dashboards

Grafana sidecar watches all namespaces for ConfigMaps labeled `grafana_dashboard: "1"`.
The following dashboards are auto-imported by k8s-monitoring:

- Kubernetes / Views / Global
- Kubernetes / Views / Nodes
- Kubernetes / Views / Namespaces
- Kubernetes / Views / Pods
