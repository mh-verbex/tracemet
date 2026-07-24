# tracemet

A self-contained observability stack for **APM, distributed tracing, metrics, and logs**,
built on the Grafana LGTM ecosystem + OpenTelemetry. One OTLP endpoint in, three
signals correlated in Grafana out.

## Stack

| Component            | Role                                                          | Port(s) |
|----------------------|---------------------------------------------------------------|---------|
| **OTel Collector**   | Single ingest point for traces/metrics/logs (OTLP)            | 4317, 4318, 8889 |
| **Prometheus**       | Metrics storage + scraping                                     | 9090 |
| **Tempo**            | Trace storage + span-metrics/service-graph generation (APM)   | 3200 |
| **Loki**             | Log storage                                                   | 3100 |
| **Promtail**         | Ships Docker container stdout logs → Loki                     | — |
| **Grafana**          | Dashboards, alerting, trace↔log↔metric correlation           | 3000 |
| **Node Exporter**    | Host CPU / memory / disk / network metrics                    | 9100 |
| **cAdvisor**         | Per-container resource metrics                                | 8080 |

## Data flow

```
your apps ──OTLP(grpc:4317 / http:4318)──► otel-collector
                                              ├─traces──► tempo
                                              ├─metrics─► (prometheus scrapes :8889)
                                              └─logs────► loki
tempo ──span metrics + service graphs (remote_write)──► prometheus
node-exporter / cadvisor ──► prometheus
promtail ──► loki
grafana ◄── prometheus + tempo + loki
```

## Quick start

```bash
cp .env.example .env          # then edit the admin password
docker compose up -d
open http://localhost:3000    # Grafana — admin / admin by default
```

The **tracemet → Overview** dashboard is provisioned automatically (RED metrics
from Tempo span-metrics + host/container metrics from node-exporter/cAdvisor).

## Sending telemetry from your app

Point your OpenTelemetry SDK / auto-instrumentation at the collector:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318   # HTTP
# or grpc: http://localhost:4317
export OTEL_SERVICE_NAME=my-service
export OTEL_RESOURCE_ATTRIBUTES=service.namespace=verbex
```

Traces, metrics, and logs exported over OTLP all land through the single
collector endpoint. Include a `trace_id` in your structured (JSON) logs and
Grafana links each log line straight to its trace in Tempo.

If your app runs in Docker, attach it to the `tracemet` network (or merge this
compose with your app's) and use `http://otel-collector:4318`.

## What you get out of the box

- **APM / service map** — Tempo's metrics-generator emits RED (rate/error/
  duration) span metrics and service graphs to Prometheus; view them in
  **Explore → Tempo → Service Graph** or the Overview dashboard.
- **Trace ↔ Logs ↔ Metrics correlation** — configured in the Grafana
  datasource provisioning (`tracesToLogs`, `tracesToMetrics`, exemplars,
  derived fields).
- **Host + container metrics** — node-exporter and cAdvisor, pre-scraped.

## Layout

```
docker-compose.yml
otel-collector/config.yaml
prometheus/prometheus.yml
tempo/tempo.yaml
loki/loki-config.yaml
promtail/promtail-config.yaml
grafana/
  provisioning/datasources/datasources.yaml
  provisioning/dashboards/dashboards.yaml
  dashboards/overview.json
.env.example
```

## Notes

- Storage is local filesystem inside named volumes — fine for a single host /
  dev / small prod. For scale, point Tempo & Loki at object storage (S3/GCS).
- Retention: Prometheus `PROMETHEUS_RETENTION` (default 15d), Tempo traces 48h
  (`compactor.compaction.block_retention` in `tempo/tempo.yaml`).
- `cadvisor` and `node-exporter` mounts assume a Linux host. On Docker Desktop
  (macOS/Windows) host metrics reflect the Linux VM, not macOS itself.
