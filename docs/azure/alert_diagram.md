# Alerting Architecture Diagram (Azure)

Alerting operates on **two independent paths**: Grafana Managed Alerting (signal-level, query-driven) and Azure-native alerting (infrastructure lifecycle, Azure Monitor / Event Grid). Both paths converge on the same notification channels.

For individual alert definitions, detection logic, and runbooks see [alerting.md](alerting.md).

---

## Path 1 — Grafana Managed Alerting (signal-level thresholds)

Grafana Alerting runs inside the Azure Managed Grafana workspace. Alert rules periodically query the backend datastores (Prometheus, Loki, Tempo) and fire when thresholds are breached.

> **Managed Grafana datasource support for alerting:** Prometheus (PromQL) and Loki (LogQL metric queries) datasources are fully supported as alert rule datasources. The Tempo datasource is trace-search only and **cannot** back alert rules directly. Tempo health alerts (A-09) must query Tempo's own Prometheus-format metrics via the Prometheus datasource instead.

```
 ┌───────────────────────────────────────────────────────────────────┐
 │  Signal Sources (continuous telemetry push)                       │
 │                                                                   │
 │   OTel Agents → Azure LB → OTel Gateway → Prometheus  (metrics)  │
 │                                          → Loki        (logs)     │
 │                                          → Tempo       (traces)   │
 └──────────────────────────┬────────────────────────────────────────┘
                            │  datasource queries (PromQL / LogQL)
                            │  note: all three backends expose Prometheus
                            │  metrics; Tempo alerting uses Prometheus DS
                            ▼
              ┌─────────────────────────────┐
              │  Azure Managed Grafana      │
              │  Alerting Engine            │
              │  (rules evaluated on AMG)   │
              └──────────────┬──────────────┘
                             │  alert state: firing
                             ▼
              ┌─────────────────────────────┐
              │   Grafana Contact Points    │
              │   ┌───────────────────┐     │
              │   │  Webhook → Action │     │
              │   │  Group → PagerDuty│     │
              │   │  / Slack / Email  │     │
              │   └───────────────────┘     │
              └─────────────────────────────┘
```

### Alerts on this path

| ID | Alert | Signal |
|----|-------|--------|
| A-02 | OTel Gateway export errors | Prometheus — `otelcol_exporter_send_failed_*` |
| A-03 | OTel Gateway queue saturation | Prometheus — `otelcol_exporter_queue_size` |
| A-05 *(partial)* | Backend storage disk saturation | Prometheus — `node_filesystem_avail_bytes` (node_exporter) |
| A-07 | Prometheus ingestion health | Prometheus — `up{job="prometheus"}`, `prometheus_remote_storage_failed_samples_total` |
| A-08 | Loki ingestion health | Prometheus — `loki_distributor_lines_received_total`, `loki_distributor_ingester_appends_failures_total` |
| A-09 | Tempo ingestion health | Prometheus — `tempo_distributor_spans_received_total`, `tempo_request_duration_seconds_count{status_code=~"5.."}` (Prometheus DS, not Tempo DS) |

---

## Path 2 — Azure-Native Alerting (infrastructure & lifecycle events)

Azure-native signals (Azure Monitor Alerts and Event Grid) fire independently of Grafana. They detect conditions where the observability platform itself may be degraded — meaning Grafana-evaluated alerts could silently be missing data.

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │  Azure Infrastructure Events                                            │
 │                                                                         │
 │   ┌─────────────────────────────────┐   ┌───────────────────────────┐  │
 │   │  Azure Monitor Alerts           │   │  Azure Monitor Alerts     │  │
 │   │  (Container App diagnostics)    │   │  (Infrastructure metrics) │  │
 │   │                                 │   │                           │  │
 │   │  ACA sidecar restart count      │   │  LB DipAvailability       │  │
 │   │  ACA sidecar crash logs         │   │  VM disk utilization      │  │
 │   │  [A-01]                         │   │  VMSS instance count      │  │
 │   │                                 │   │  [A-04] [A-05] [A-06]     │  │
 │   └──────────────┬──────────────────┘   └────────────┬──────────────┘  │
 └──────────────────┼──────────────────────────────────┼──────────────────┘
                    │                                  │
                    │  Action Group                    │  Action Group
                    ▼                                  ▼
              ┌─────────────────────────────────────────────┐
              │   Azure Monitor Action Groups               │
              │   (observability-alerts-critical/warning)    │
              └──────────────────────┬──────────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                 ▼
             ┌───────────┐   ┌───────────┐   ┌──────────────┐
             │ PagerDuty │   │   Slack   │   │    Email     │
             │ (webhook) │   │ (webhook) │   │              │
             └───────────┘   └───────────┘   └──────────────┘
```

### Alerts on this path

| ID | Alert | Azure Source |
|----|-------|-------------|
| A-01 | ACA sidecar telemetry disruption | Azure Monitor — Container App restart count / crash logs |
| A-04 | Azure LB backend pool unhealthy | Azure Monitor Alert — `DipAvailability` metric |
| A-05 *(partial)* | Backend storage disk saturation | Azure Monitor Alert — AMA `disk_used_percent` |
| A-06 | OTel Gateway VMSS below instance floor | Azure Monitor Alert — VMSS instance count |

> **Why two paths matter:** A-01, A-04, and A-06 describe conditions where the platform itself is impaired. If the OTel Gateway is down or a sidecar is unstable, Prometheus may not be receiving data — so Grafana-only alerting would be blind. The Azure-native path fires independently and remains reliable even during partial platform outages.

---

## Combined View — Both Paths to Notification Channels

```
  Signal-level thresholds                Infrastructure & lifecycle
  ─────────────────────────              ──────────────────────────
  Prometheus / Loki / Tempo              Azure Monitor Alerts
       │                                        │
       │  datasource queries                    │  metric/log alert
       ▼                                        ▼
  Grafana Alerting Engine            Action Groups
       │                            (observability-alerts-*)
       │  contact points                        │
       │  (webhook → Action Group)              │  receivers
       └──────────────────┬─────────────────────┘
                          ▼
             ┌────────────────────────┐
             │  Notification Channels │
             │  PagerDuty (webhook)   │
             │  Slack (webhook)       │
             │  Email                 │
             └────────────────────────┘
```

> **Grafana → Action Group integration:** Grafana contact points can fire webhooks directly to Azure Monitor Action Groups, unifying the notification delivery path. Alternatively, Grafana can call PagerDuty/Slack webhook URLs directly without going through Action Groups. The recommended pattern is to route Grafana alerts through Action Groups so both paths share the same notification configuration and deduplication logic.

---

## Scope Notes

- **What is shown here:** the alert *delivery path* — how firing conditions reach operators.
- **What is not shown here:** individual alert rules, PromQL/LogQL expressions, severity routing, silencing, and inhibition. Those are documented per-alert in [alerting.md](alerting.md).
- A-05 (disk saturation) spans both paths: `node_exporter` data is evaluated by Grafana Alerting; AMA `disk_used_percent` triggers an independent Azure Monitor Alert as a backup.

### Key Differences from AWS Alert Architecture

| Aspect | AWS Design | Azure Design |
|--------|-----------|--------------|
| Infrastructure event bus | Amazon EventBridge | Azure Monitor Alerts (no Event Grid needed for these use cases) |
| Notification routing | Amazon SNS topics | Azure Monitor Action Groups |
| Container sidecar detection | EventBridge rule on ECS Task State Change (sidecar stopped while task running) | Azure Monitor metric/log alert on ACA restart count and crash logs |
| LB health | CloudWatch `HealthyHostCount` alarm | Azure Monitor `DipAvailability` metric alert |
| Auto-scaling group health | CloudWatch `GroupInServiceInstances` alarm | Azure Monitor VMSS instance count metric alert |
| Disk alerting (native path) | CloudWatch Agent `disk_used_percent` alarm | Azure Monitor Agent (AMA) `disk/used_percent` alert |
| Grafana → Notification | Grafana contact point → SNS topic | Grafana contact point → webhook → Action Group |
