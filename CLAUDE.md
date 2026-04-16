# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Release workflow

**Every commit that modifies chart content must bump the version in `charts/k8s-grafana-stack/Chart.yaml` before being committed.** The GitHub Actions workflow (`release.yaml`) triggers on every push to `main` and reads the version directly from `Chart.yaml` to package the chart, push it to GHCR as an OCI artifact, and create a GitHub release. If the version is not bumped, the release step will silently skip (`|| echo "Release already exists, skipping"`) and the new content will never be published.

Version format is `MAJOR.MINOR.PATCH` — increment `PATCH` for fixes and non-breaking changes, `MINOR` for new features.

Commit order per change set:
1. Content changes (templates, values, etc.)
2. `chore: bump to X.Y.Z` — version bump in `Chart.yaml`

Or combine both in a single commit when the change is atomic.

## Architecture

This repository is a single Helm chart (`charts/k8s-grafana-stack`) that composes eight upstream sub-charts into a self-hosted Kubernetes observability stack. The chart has no application code — only Helm templates, values, and two custom templates.

### Custom templates

The chart adds exactly two files on top of the sub-charts:

- `templates/alloy-config.yaml` — a `ConfigMap` containing the full Alloy pipeline in Alloy's River syntax. It is conditionally rendered based on `values.yaml` flags (`prometheus.enabled`, `loki.enabled`, `collection.*`). Alloy is told to use this ConfigMap via `alloy.alloy.configMap.create: false` + name/key references.
- `templates/alloy-rbac.yaml` — `ClusterRole` + `ClusterRoleBinding` for Alloy's Kubernetes API access (pod discovery, node metrics, events). The sub-chart's own RBAC is disabled (`alloy.rbac.create: false`).

### Alloy pipeline structure

The Alloy config is built from conditional blocks that are stitched together at Helm render time:

```
prometheus.scrape (kube-state-metrics, node-exporter, kubelet, cadvisor, annotated pods)
    └─► prometheus.remote_write.default → Prometheus

loki.source.kubernetes (pod logs) ──[optional loki.process]──► loki.write.default → Loki
loki.source.kubernetes_events ──────────────────────────────► loki.write.default

otelcol.receiver.otlp
    └─► otelcol.processor.memory_limiter
            └─► otelcol.processor.batch
                    ├─► otelcol.exporter.otlp.tempo      → Tempo
                    ├─► otelcol.exporter.prometheus       → prometheus.remote_write.default
                    └─► otelcol.exporter.loki             → loki.write.default
```

### Sub-chart value passing

Sub-chart values are namespaced by the dependency name in `values.yaml`. The Alloy sub-chart (named `alloy`) has two distinct top-level keys in its own values: `alloy` (config map, resources, ports) and `controller` (type, replicas). In the parent chart these map to:

- `alloy.alloy.*` → sub-chart's `alloy.*` section
- `alloy.controller.*` → sub-chart's `controller.*` section (type, replicas)

Putting `controller` under `alloy.alloy.controller` is a silent no-op — the sub-chart ignores it.

### Collection flags

`collection.*` flags in `values.yaml` gate entire pipeline blocks in `alloy-config.yaml`:

| Flag | Effect |
|------|--------|
| `collection.scraping.enabled` | Annotation-based pod scraping |
| `collection.logs.stdout.enabled` | Pod stdout + cluster events → Loki |
| `collection.otlp.traces.enabled` | OTLP receiver traces → Tempo |
| `collection.otlp.metrics.enabled` | OTLP receiver metrics → Prometheus |
| `collection.otlp.logs.enabled` | OTLP receiver logs → Loki |

The OTLP receiver block (`otelcol.receiver.otlp`) is only rendered when at least one `collection.otlp.*` flag is true. The memory_limiter and batch processors follow the same condition.
