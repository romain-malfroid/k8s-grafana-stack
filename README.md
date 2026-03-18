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

## Prerequisites

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm dependency update
```

---

## Install

### Any Kubernetes distribution

```bash
helm install k8s-grafana-stack . -f values.yaml -n monitoring --create-namespace
```

### k3s (Traefik Gateway API)

```bash
helm install k8s-grafana-stack . -f values.yaml -f values-k3s.yaml -f your-overrides.yaml -n monitoring --create-namespace
```

`values-k3s.yaml` fixes k3s-specific issues (node-exporter port conflicts, Alloy clustering).
Add a third `-f your-overrides.yaml` for your deployment-specific values (gateway name, domain).

### Upgrade

```bash
helm upgrade k8s-grafana-stack . -f values.yaml [-f values-k3s.yaml] [-f your-overrides.yaml] -n monitoring
```

---

## Configuration

All configuration is in `values.yaml`. Key sections:

| Section | Key settings |
|---------|-------------|
| `k8s-monitoring` | `cluster.name`, collection destinations, node-exporter |
| `prometheus.server` | `retention`, `persistentVolume.size` |
| `loki` | `deploymentMode` (SingleBinary), MinIO storage |
| `tempo` | `retention`, OTLP receivers |
| `grafana` | `adminPassword`, datasources, persistence |

### Platform overrides file (example)

```yaml
# your-overrides.yaml — deployment-specific values, not committed to the chart repo
traefik:
  enabled: true
  gatewayName: "my-gateway"
  gatewayNamespace: "default"

grafana:
  grafana.ini:
    server:
      root_url: "http://my-domain.com/grafana"
      serve_from_sub_path: true
```

### Grafana default credentials

`admin` / `admin` — change via `grafana.adminPassword` or after first login.

---

## Dashboards

Grafana sidecar watches all namespaces for ConfigMaps labeled `grafana_dashboard: "1"`.
The following dashboards are auto-imported by k8s-monitoring:

- Kubernetes / Views / Global
- Kubernetes / Views / Nodes
- Kubernetes / Views / Namespaces
- Kubernetes / Views / Pods
