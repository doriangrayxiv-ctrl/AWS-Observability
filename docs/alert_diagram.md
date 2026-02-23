# Alerting Architecture Diagram

Alerting operates on **two independent paths**: Grafana Managed Alerting (signal-level, query-driven) and AWS-native alerting (infrastructure lifecycle, CloudWatch / EventBridge). Both paths converge on the same notification channels.

For individual alert definitions, detection logic, and runbooks see [alerting.md](alerting.md).

---

## Path 1 — Grafana Managed Alerting (signal-level thresholds)

Grafana Alerting runs inside the AMG workspace. Alert rules periodically query the backend datastores (Prometheus, Loki, Tempo) and fire when thresholds are breached.

> **AMG datasource support for alerting:** Prometheus (PromQL) and Loki (LogQL metric queries) datasources are fully supported as alert rule datasources in AMG. The Tempo datasource is trace-search only and **cannot** back alert rules directly. Tempo health alerts (A-09) must query Tempo's own Prometheus-format metrics via the Prometheus datasource instead.

```
 ┌───────────────────────────────────────────────────────────────────┐
 │  Signal Sources (continuous telemetry push)                       │
 │                                                                   │
 │   OTel Agents → NLB → OTel Gateway → Prometheus  (metrics)       │
 │                                    → Loki        (logs)           │
 │                                    → Tempo       (traces)         │
 └──────────────────────────┬────────────────────────────────────────┘
                            │  datasource queries (PromQL / LogQL)
                            │  note: all three backends expose Prometheus
                            │  metrics; Tempo alerting uses Prometheus DS
                            ▼
              ┌─────────────────────────────┐
              │   AWS Managed Grafana       │
              │   Alerting Engine           │
              │   (rules evaluated on AMG)  │
              └──────────────┬──────────────┘
                             │  alert state: firing
                             ▼
              ┌─────────────────────────────┐
              │   Grafana Contact Points    │
              │   ┌───────────────────┐     │
              │   │  PagerDuty        │     │
              │   │  Slack            │     │
              │   │  Email            │     │
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
| A-08 | Loki ingestion health | Prometheus — `loki_distributor_lines_received_total`, `loki_request_duration_seconds_count{status_code=~"5.."}` |
| A-09 | Tempo ingestion health | Prometheus — `tempo_distributor_spans_received_total`, `tempo_request_error_rate` (Prometheus DS, not Tempo DS) |

---

## Path 2 — AWS-Native Alerting (infrastructure & lifecycle events)

AWS-native signals (CloudWatch Alarms and EventBridge rules) fire independently of Grafana. They detect conditions where the observability platform itself may be degraded — meaning Grafana-evaluated alerts could silently be missing data.

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │  AWS Infrastructure Events                                              │
 │                                                                         │
 │   ┌─────────────────────────────────┐   ┌───────────────────────────┐  │
 │   │  Amazon EventBridge             │   │  Amazon CloudWatch Alarms │  │
 │   │                                 │   │                           │  │
 │   │  Rule: ECS Task State Change    │   │  NLB UnhealthyHostCount   │  │
 │   │  (otelcol-contrib STOPPED while │   │  EBS disk utilization     │  │
 │   │   task still RUNNING)   [A-01]  │   │  ASG min instance breach  │  │
 │   │                                 │   │  [A-04] [A-05] [A-06]     │  │
 │   └──────────────┬──────────────────┘   └────────────┬──────────────┘  │
 └──────────────────┼──────────────────────────────────┼──────────────────┘
                    │                                  │
                    │  SNS publish                     │  SNS publish
                    ▼                                  ▼
              ┌─────────────────────────────────────────────┐
              │   SNS Topic  (observability-alerts)         │
              └──────────────────────┬──────────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                 ▼
             ┌───────────┐   ┌───────────┐   ┌──────────────┐
             │ PagerDuty │   │   Slack   │   │    Email     │
             └───────────┘   └───────────┘   └──────────────┘
```

### Alerts on this path

| ID | Alert | AWS Source |
|----|-------|-----------|
| A-01 | ECS sidecar silent telemetry blackout | EventBridge — ECS Task State Change |
| A-04 | NLB target group unhealthy hosts | CloudWatch Alarm — `UnHealthyHostCount` |
| A-05 *(partial)* | Backend storage disk saturation | CloudWatch Alarm — EBS `VolumeUtilization` |
| A-06 | OTel Gateway ASG below instance floor | CloudWatch Alarm — EC2 Auto Scaling |

> **Why two paths matter:** A-01, A-04, and A-06 describe conditions where the platform itself is impaired. If the OTel Gateway is down or a sidecar is dead, Prometheus may not be receiving data — so Grafana-only alerting would be blind. The AWS-native path fires independently and remains reliable even during partial platform outages.

---

## Combined View — Both Paths to Notification Channels

```
  Signal-level thresholds                Infrastructure & lifecycle
  ─────────────────────────              ──────────────────────────
  Prometheus / Loki / Tempo              EventBridge / CloudWatch
       │                                        │
       │  datasource queries                    │  alarm / event
       ▼                                        ▼
  AMG Alerting Engine               SNS Topic (observability-alerts)
       │                                        │
       │  contact points                        │  subscriptions
       └──────────────────┬─────────────────────┘
                          ▼
             ┌────────────────────────┐
             │  Notification Channels │
             │  PagerDuty             │
             │  Slack                 │
             │  Email                 │
             └────────────────────────┘
```

---

## Scope Notes

- **What is shown here:** the alert *delivery path* — how firing conditions reach operators.
- **What is not shown here:** individual alert rules, PromQL/LogQL expressions, severity routing, silencing, and inhibition. Those are documented per-alert in [alerting.md](alerting.md).
- A-05 (disk saturation) spans both paths: `node_exporter` data is evaluated by Grafana Alerting; EBS CloudWatch metrics trigger an independent CloudWatch Alarm as a backup.
