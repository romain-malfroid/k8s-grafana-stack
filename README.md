# 📊 k8s-grafana-stack

> Helm chart for a complete Kubernetes observability stack — metrics, logs and traces, collected by Alloy and visualized in Grafana.

![Architecture](https://raw.githubusercontent.com/romain-malfroid/k8s-grafana-stack/main/docs/Alloy.png)

## ✨ What's included

- 🔍 **Auto-discovery** — Alloy scrapes cluster metrics and pod logs automatically
- 📦 **Pre-configured datasources** — Prometheus, Loki and Tempo ready to use in Grafana
- 📈 **Pre-installed dashboards** — Kubernetes views for nodes, pods, namespaces and cluster
- 🎛️ **Modular** — disable metrics, logs or traces independently

---

## 🚀 Install

The chart is distributed as an OCI artifact via GitHub Container Registry — no `helm repo add` needed.
It is recommended to install in a dedicated `monitoring` namespace.

```bash
helm install k8s-grafana-stack oci://ghcr.io/romain-malfroid/helm-charts/k8s-grafana-stack \
  -n monitoring --create-namespace
```

Pin a specific version with `--version 0.2.18`.

Access Grafana at http://localhost:3000 (admin / admin):

```bash
kubectl port-forward svc/k8s-grafana-stack-grafana 3000:80 -n monitoring
```

### 🌐 With ingress

To expose Grafana via your ingress controller. A commented example is available in [`charts/k8s-grafana-stack/values.yaml`](charts/k8s-grafana-stack/values.yaml).

Works with any ingress controller (nginx, Traefik, etc.) including Gateway API implementations.

```yaml
grafana:
  ingress:
    enabled: true
    ingressClassName: nginx     # or traefik, or any other ingress controller
    hosts:
      - your-domain
    path: /grafana
  grafana.ini:
    server:
      root_url: "http://your-domain/grafana"
      serve_from_sub_path: true
```

```bash
helm install k8s-grafana-stack oci://ghcr.io/romain-malfroid/helm-charts/k8s-grafana-stack \
  -f values.yaml -n monitoring --create-namespace
```

### 🔧 k3s

Uncomment two lines in your `values.yaml` to avoid port conflicts with k3s internal processes:

```yaml
prometheus-node-exporter:
  hostNetwork: false
  hostPID: false
```

> ⚠️ This trades a small number of low-level host metrics for a working deployment — recommended approach for k3s.

### ⬆️ Upgrade

```bash
helm upgrade k8s-grafana-stack oci://ghcr.io/romain-malfroid/helm-charts/k8s-grafana-stack \
  [-f values.yaml] -n monitoring
```

---

## ⚙️ Configuration

### 🔌 Enable / disable components

By default all components are enabled. Set `enabled: false` to disable a component — this removes both the storage layer and the corresponding Alloy collection pipeline:

```yaml
prometheus:
  enabled: false  # disables metrics (Prometheus, node-exporter, kube-state-metrics, Alertmanager)

loki:
  enabled: false  # disables log storage and collection

tempo:
  enabled: false  # disables trace storage and collection
```
### 🛠️ Common overrides

| Key | Description | Default |
|-----|-------------|---------|
| `clusterName` | Cluster name shown in dashboards | `"local"` |
| `grafana.adminPassword` | Grafana admin password | `admin` |
| `prometheus.server.retention` | Metrics retention period | `7d` |
| `tempo.tempo.retention` | Traces retention period | `24h` |

### 📡 Collection modes

The chart supports two collection modes that can be combined:

| Mode | Metrics | Logs | Traces |
|------|---------|------|--------|
| **Pull (default)** | Alloy scrapes `/metrics` via annotations | Alloy reads pod stdout | OTLP push |
| **OTLP push** | Apps push via OTLP | Apps push via OTLP | OTLP push |
| **Hybrid** | Both | Both | OTLP push |

> ⚠️ In hybrid mode, an app that both exposes `/metrics` AND pushes OTLP metrics will have its metrics duplicated in Prometheus.

**Pull mode** (default — no config needed):

```yaml
collection:
  scraping:
    enabled: true   # default
  otlp:
    metrics:
      enabled: false  # default
    logs:
      enabled: false  # default
    traces:
      enabled: true   # default
```

**OTLP mode** (apps push all signals):

```yaml
collection:
  scraping:
    enabled: false
  otlp:
    metrics:
      enabled: true
    logs:
      enabled: true
    traces:
      enabled: true
```

Apps send to Alloy at:
- gRPC: `k8s-grafana-stack-alloy.monitoring.svc:4317`
- HTTP: `k8s-grafana-stack-alloy.monitoring.svc:4318`

**Pull — scrape annotations:**

```yaml
# In your Deployment / StatefulSet, under spec.template.metadata
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"     # port your app exposes metrics on
    prometheus.io/path: "/metrics" # optional — use /actuator/prometheus for Spring Boot
```

### 🏷️ Kubernetes labeling conventions

For proper grouping across metrics, logs and traces, follow the official Kubernetes labeling conventions:

| Label | Example | Description |
|-------|---------|-------------|
| `app.kubernetes.io/name` | `api-gateway` | Application name |
| `app.kubernetes.io/instance` | `api-gateway-prod` | Unique instance |
| `app.kubernetes.io/version` | `1.2.3` | Version |
| `app.kubernetes.io/component` | `backend` | Component role |
| `app.kubernetes.io/part-of` | `my-app` | Parent application |
| `app.kubernetes.io/managed-by` | `helm` | Management tool |

> ⚠️ Non-standard labels may result in incomplete or poorly grouped data in Grafana.

### 🧩 Advanced Alloy configuration

**`extraAlloyConfig`** — append custom Alloy config blocks (custom scrape jobs, etc.):

```yaml
extraAlloyConfig: |
  prometheus.scrape "my_app" {
    targets = [{ __address__ = "my-service.default.svc:9090" }]
    forward_to = [prometheus.remote_write.default.receiver]
  }
```

**`logsProcessStages`** — inject pipeline stages between log collection and Loki (parse log levels, extract fields, etc.):

```yaml
logsProcessStages: |
  stage.regex {
    expression = `(?i)\b(?P<level>TRACE|DEBUG|INFO|WARN|ERROR|FATAL)(?:ING)?\b`
  }
  stage.labels {
    values = { level = "level" }
  }
```

See the [Alloy documentation](https://grafana.com/docs/alloy/latest/reference/components/) for all available components.

---

## 📦 Stack

| Component | Chart | Version | Role |
|-----------|-------|---------|------|
| **Alloy** | `grafana/alloy` | 1.6.2 | Collector |
| **kube-state-metrics** | `prometheus-community/kube-state-metrics` | 7.2.1 | Kubernetes object metrics |
| **node-exporter** | `prometheus-community/prometheus-node-exporter` | 4.52.1 | Node-level metrics |
| **Prometheus** | `prometheus-community/prometheus` | 28.13.0 | Metrics storage |
| **Alertmanager** | `prometheus-community/alertmanager` | 1.33.1 | Alerting |
| **Loki** | `grafana/loki` | 6.55.0 | Log storage |
| **Tempo** | `grafana/tempo` | 1.24.4 | Trace storage |
| **Grafana** | `grafana/grafana` | 10.5.15 | Visualization |

> 💡 **Metrics only?** Use [`kube-prometheus-stack`](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) — it bundles Prometheus Operator, Grafana, node-exporter and kube-state-metrics in a single chart.
>
> ☁️ **Grafana Cloud?** Use [`k8s-monitoring`](https://github.com/grafana/k8s-monitoring-helm) — Grafana's official chart for managed Kubernetes sending telemetry to Grafana Cloud. This chart is the right choice when you self-host the full MELT stack.

---

## 📊 Dashboards

The following dashboards are **automatically installed** at startup:

| Dashboard | ID |
|-----------|----|
| Kubernetes / Views / Global | [15757](https://grafana.com/grafana/dashboards/15757) |
| Kubernetes / Views / Namespaces | [15758](https://grafana.com/grafana/dashboards/15758) |
| Kubernetes / Views / Nodes | [15759](https://grafana.com/grafana/dashboards/15759) |
| Kubernetes / Views / Pods | [15760](https://grafana.com/grafana/dashboards/15760) |

To disable a dashboard:

```yaml
grafana:
  dashboards:
    kubernetes:
      k8s-views-nodes: null
```
