# Alerting Strategy (Azure)

Alerting for the observability platform is implemented at multiple layers: **Azure-native** (Azure Monitor Alerts + Azure Event Grid) for infrastructure and container lifecycle events, and **Grafana Managed Alerting** (backed by Prometheus, Loki query, or Azure Monitor datasource) for signal-level thresholds and platform health.

> **Backend health coverage (A-07 / A-08 / A-09):** Prometheus, Loki, and Tempo each expose Prometheus-format self-metrics (ingestion counters, error rates, internal health). Azure Managed Grafana alert rules query these via the **Prometheus datasource** (PromQL). The Loki datasource also supports LogQL metric queries for alerting. The **Tempo datasource cannot back Grafana alert rules** — Tempo health alerts use the Prometheus datasource scraping Tempo's `/metrics` endpoint instead.

For a visual overview of how both paths deliver notifications to operators, see [alert_diagram.md](alert_diagram.md).

This document records each discrete alerting mechanism: what it detects, how it is implemented, and expected operator response.

---

## Index

| ID | Alert | Severity | Signal Source | Status |
|----|-------|----------|---------------|--------|
| [A-01](#a-01-aca-sidecar-telemetry-disruption) | ACA sidecar telemetry disruption | **Warning** | Azure Monitor (Container App diagnostics) | Active |
| [A-02](#a-02-otel-gateway-export-errors) | OTel Gateway export errors | **Critical** | Prometheus (otelcol internal metrics) | Active |
| [A-03](#a-03-otel-gateway-queue-saturation) | OTel Gateway queue saturation | **Warning → Critical** | Prometheus (otelcol internal metrics) | Active |
| [A-04](#a-04-azure-lb-backend-pool-unhealthy) | Azure LB backend pool unhealthy | **Critical** | Azure Monitor (LB health probe status) | Active |
| [A-05](#a-05-backend-storage-disk-saturation) | Backend storage disk saturation (Prometheus / Loki / Tempo) | **Warning → Critical** | Azure Monitor (disk utilization) + Prometheus (node_exporter) | Active |
| [A-06](#a-06-otel-gateway-vmss-instance-count-below-floor) | OTel Gateway VMSS instance count below floor | **Critical** | Azure Monitor (VMSS instance count) | Active |
| [A-07](#a-07-prometheus-ingestion-health) | Prometheus ingestion health | **Critical** | Prometheus (self-metrics via Prometheus DS on Managed Grafana) | Active |
| [A-08](#a-08-loki-ingestion-health) | Loki ingestion health | **Critical** | Prometheus (Loki self-metrics via Prometheus DS on Managed Grafana) | Active |
| [A-09](#a-09-tempo-ingestion-health) | Tempo ingestion health | **Warning → Critical** | Prometheus (Tempo self-metrics via Prometheus DS on Managed Grafana) | Active |

---

## A-01: ACA Sidecar Telemetry Disruption

### Problem

The `otelcol-contrib` sidecar container in every Azure Container App runs alongside the application container. Unlike ECS Fargate where sidecars can be marked `essential: false` (allowing silent telemetry blackouts), ACA sidecar crashes trigger the container app's restart policy. With `OnFailure` or `Always` restart policy (recommended), the sidecar auto-restarts on crash — but telemetry is lost during the restart window.

The primary risk is not permanent blackout (ACA self-heals) but **repeated restarts** indicating a configuration or resource problem, and the brief telemetry gaps during each restart cycle.

### Detection Mechanism

Azure Container Apps publishes container-level diagnostics to Azure Monitor. When the `otelcol-contrib` sidecar container restarts, the following signals are available:

1. **Azure Monitor Metrics** — `ContainerAppReplicaRestartCount` metric tracks container restarts within a replica. A sustained non-zero rate for the sidecar container indicates instability.
2. **Azure Monitor Logs** — Container App system logs (sent to Log Analytics workspace when diagnostic settings are enabled) record container crash events, exit codes, and restart reasons.

### Implementation

#### 1. Azure Monitor Alert Rule (Metric Alert)

Create a metric alert on the Container App resource:

```hcl
resource "azurerm_monitor_metric_alert" "aca_sidecar_restarts" {
  name                = "aca-otelcol-sidecar-restart-warning"
  resource_group_name = azurerm_resource_group.observability.name
  scopes              = [azurerm_container_app_environment.main.id]
  description         = "OTel sidecar container is restarting repeatedly — telemetry gaps occurring"
  severity            = 2  # Warning
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.App/containerApps"
    metric_name      = "RestartCount"
    aggregation      = "Total"
    operator         = "GreaterThan"
    threshold        = 3

    dimension {
      name     = "ContainerName"
      operator = "Include"
      values   = ["otelcol-contrib"]
    }
  }

  action {
    action_group_id = azurerm_monitor_action_group.observability_warning.id
  }
}
```

#### 2. Azure Monitor Log Alert (Log Analytics query)

For more detailed detection including exit codes:

```hcl
resource "azurerm_monitor_scheduled_query_rules_alert_v2" "aca_sidecar_crash" {
  name                = "aca-otelcol-sidecar-crash"
  resource_group_name = azurerm_resource_group.observability.name
  location            = azurerm_resource_group.observability.location
  scopes              = [azurerm_log_analytics_workspace.observability.id]
  description         = "OTel sidecar container crashed in ACA — check exit code and logs"
  severity            = 1  # Error

  criteria {
    query = <<-KQL
      ContainerAppSystemLogs_CL
      | where ContainerName_s == "otelcol-contrib"
      | where Reason_s == "CrashLoopBackOff" or Reason_s == "Error" or Reason_s == "OOMKilled"
      | summarize CrashCount = count() by ContainerAppName_s, Reason_s, bin(TimeGenerated, 5m)
      | where CrashCount > 0
    KQL

    time_aggregation_method = "Count"
    operator                = "GreaterThan"
    threshold               = 0

    failing_periods {
      minimum_failing_periods_to_trigger_alert = 1
      number_of_evaluation_periods             = 1
    }
  }

  window_duration      = "PT5M"
  evaluation_frequency = "PT5M"

  action {
    action_groups = [azurerm_monitor_action_group.observability_critical.id]
  }
}
```

#### 3. Action Group → Notification Channels

Route the Action Group to:

| Channel | Integration |
|---------|-------------|
| PagerDuty | Action Group → Webhook → PagerDuty Events API v2 |
| Slack `#observability-alerts` | Action Group → Webhook → Slack Incoming Webhook |
| Email (on-call DL) | Action Group → Email receiver |

#### 4. Grafana Alert (platform health dashboard)

Add a panel to the **OTel Platform Health** dashboard that queries ACA restart metrics via the Azure Monitor datasource, with a Grafana alert rule configured to fire when sidecar restarts exceed the threshold.

### Operator Response

| Condition | Action |
|-----------|--------|
| Single restart, non-repeating | ACA auto-restarted the sidecar. Verify telemetry resumed in Grafana. No action if stable. |
| Repeated restarts (CrashLoopBackOff) | Check sidecar container logs in Log Analytics. Common causes: invalid YAML config, unreachable Gateway endpoint, TLS cert mismatch, OOM. |
| OOMKilled | Increase sidecar memory allocation in the container app definition (from 0.5 Gi → 1 Gi). Review batch size and memory limiter thresholds in the sidecar collector config. |
| All container apps affected | Likely a config push error. Roll back the sidecar config in Azure Key Vault / App Configuration. Redeploy container app revisions with the previous config version. |

### Comparison with AWS (A-01)

The AWS ECS design used EventBridge to detect tasks where the sidecar stopped but the app continued running (`essential: false`). In ACA, the failure model is fundamentally different: sidecar crashes trigger automatic restarts (no permanent silent blackout), so the alert focuses on **restart frequency** rather than stopped-sidecar detection. The risk is repeated brief gaps, not indefinite blindness.

---

## A-02: OTel Gateway Export Errors

### Problem

The OTel Gateway Collectors export processed telemetry to Prometheus (remote-write), Loki, and Tempo. If any exporter begins failing — due to a backend being down, a misconfigured endpoint, authentication failure, or network disruption — telemetry is silently dropped after the retry budget is exhausted.

### Detection Mechanism

`otelcol-contrib` exposes Prometheus-format internal metrics on its metrics endpoint (default `:8888`). Key metrics:

| Metric | Description |
|--------|-------------|
| `otelcol_exporter_send_failed_spans_total` | Spans permanently dropped by an exporter |
| `otelcol_exporter_send_failed_metric_points_total` | Metric points permanently dropped |
| `otelcol_exporter_send_failed_log_records_total` | Log records permanently dropped |
| `otelcol_exporter_enqueue_failed_spans_total` | Spans dropped due to queue full |
| `otelcol_exporter_enqueue_failed_metric_points_total` | Metric points dropped due to queue full |
| `otelcol_exporter_enqueue_failed_log_records_total` | Log records dropped due to queue full |

Prometheus scrapes these metrics from each Gateway instance via the VMSS instance discovery. Grafana evaluates alert rules directly against the Prometheus datasource.

### Implementation

#### 1. Prometheus Scrape Config (Gateway instances)

Add a scrape job to the Prometheus server for the Gateway VMSS. Use `azure_sd_configs` for service discovery:

```yaml
scrape_configs:
  - job_name: otelcol_gateway
    azure_sd_configs:
      - subscription_id: "<subscription-id>"
        resource_group: "observability-rg"
        port: 8888
    relabel_configs:
      - source_labels: [__meta_azure_machine_name]
        target_label: instance
      - source_labels: [__meta_azure_machine_tag_Environment]
        target_label: environment
```

#### 2. Grafana Alert Rules

```yaml
- alert: OtelGatewayExportFailed
  expr: |
    sum by (exporter, instance) (
      rate(otelcol_exporter_send_failed_spans_total[5m]) +
      rate(otelcol_exporter_send_failed_metric_points_total[5m]) +
      rate(otelcol_exporter_send_failed_log_records_total[5m])
    ) > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "OTel Gateway exporter {{ $labels.exporter }} on {{ $labels.instance }} is dropping telemetry"
    description: "Export failures detected. Backend for exporter {{ $labels.exporter }} may be down or misconfigured."
    runbook: "https://wiki.company.com/observability/runbooks/gateway-export-failure"

- alert: OtelGatewayEnqueueFailed
  expr: |
    sum by (exporter, instance) (
      rate(otelcol_exporter_enqueue_failed_spans_total[5m]) +
      rate(otelcol_exporter_enqueue_failed_metric_points_total[5m]) +
      rate(otelcol_exporter_enqueue_failed_log_records_total[5m])
    ) > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "OTel Gateway exporter {{ $labels.exporter }} queue is full on {{ $labels.instance }}"
    description: "Telemetry is being dropped because the exporter queue is at capacity. Backend is likely degraded."
    runbook: "https://wiki.company.com/observability/runbooks/gateway-queue-full"
```

#### 3. Notification Channel

Route fired alerts through Grafana's notification policy → **contact point: Webhook** → Action Group → PagerDuty / Slack `#observability-alerts` / email.

### Operator Response

| Condition | Action |
|-----------|--------|
| Export failures for one exporter | Check the specific backend (Prometheus / Loki / Tempo): `systemctl status prometheus` or VM disk utilization. |
| All exporters failing simultaneously | Verify Gateway instance connectivity to backends. Check NSG rules; confirm target VMs are running. |
| Enqueue failures | Backend is not recovering fast enough. Scale the backend vertically, or temporarily reduce ingest rate at the agent level by increasing batch intervals. |
| After recovery | Confirm failure counter reset; verify telemetry resumed in Grafana Explorer for each signal. |

---

## A-03: OTel Gateway Queue Saturation

### Problem

The OTel Gateway uses an in-memory queue (`queue_size`) to buffer telemetry before export. If a backend is slow or down, the queue fills up. Once at capacity, incoming telemetry is dropped.

### Detection Mechanism

| Metric | Description |
|--------|-------------|
| `otelcol_exporter_queue_size` | Current number of items in the exporter queue |
| `otelcol_exporter_queue_capacity` | Maximum queue capacity (items) |

Queue saturation ratio = `otelcol_exporter_queue_size / otelcol_exporter_queue_capacity`.

### Implementation

```yaml
- alert: OtelGatewayQueueWarning
  expr: |
    (
      otelcol_exporter_queue_size
      /
      otelcol_exporter_queue_capacity
    ) > 0.70
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "OTel Gateway exporter queue is >70% full on {{ $labels.instance }}"
    description: "Exporter: {{ $labels.exporter }}. Queue at {{ $value | humanizePercentage }}. Backend may be degraded."
    runbook: "https://wiki.company.com/observability/runbooks/gateway-queue-saturation"

- alert: OtelGatewayQueueCritical
  expr: |
    (
      otelcol_exporter_queue_size
      /
      otelcol_exporter_queue_capacity
    ) > 0.90
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "OTel Gateway exporter queue is >90% full on {{ $labels.instance }} — data loss imminent"
    description: "Exporter: {{ $labels.exporter }}. Queue at {{ $value | humanizePercentage }}. Telemetry will be dropped if backend does not recover."
    runbook: "https://wiki.company.com/observability/runbooks/gateway-queue-saturation"
```

### Operator Response

| Condition | Action |
|-----------|--------|
| Warning (70–90%) | Investigate backend health (see A-02 response). Check if a VMSS scale-out event is needed. |
| Critical (>90%) | Immediate backend investigation. Consider increasing `queue_size` in the Gateway config as a temporary measure. |
| Queue full + export errors (A-02 also firing) | Backend is down or unreachable; escalate to on-call. Data loss is occurring. |

---

## A-04: Azure LB Backend Pool Unhealthy

### Problem

The Azure Load Balancer Standard fronts the OTel Gateway VMSS. If all Gateway instances become unhealthy (fail health probes), inbound OTLP traffic from 1000+ collector agents is silently dropped at the LB — no back-pressure, no error returned to the push source. The Azure Monitor metric `HealthProbeStatus` and `DipAvailability` (Data path availability) are the authoritative signals.

### Detection Mechanism

Azure Monitor metric `DipAvailability` on the LB resource. The VMSS minimum is 2 instances; a `DipAvailability` below 100% indicates at least one backend is unhealthy. A value of 0% is a full platform outage.

### Implementation

```hcl
resource "azurerm_monitor_metric_alert" "lb_unhealthy_warning" {
  name                = "lb-otel-gateway-backend-degraded"
  resource_group_name = azurerm_resource_group.observability.name
  scopes              = [azurerm_lb.otel_gateway.id]
  description         = "Azure LB backend pool has unhealthy instances — Gateway capacity is degraded"
  severity            = 2  # Warning
  frequency           = "PT1M"
  window_size         = "PT5M"

  criteria {
    metric_namespace = "Microsoft.Network/loadBalancers"
    metric_name      = "DipAvailability"
    aggregation      = "Average"
    operator         = "LessThan"
    threshold        = 100
  }

  action {
    action_group_id = azurerm_monitor_action_group.observability_warning.id
  }
}

resource "azurerm_monitor_metric_alert" "lb_no_healthy_backends" {
  name                = "lb-otel-gateway-no-healthy-backends-CRITICAL"
  resource_group_name = azurerm_resource_group.observability.name
  scopes              = [azurerm_lb.otel_gateway.id]
  description         = "CRITICAL: Azure LB has zero healthy Gateway backends — all inbound telemetry is being dropped"
  severity            = 0  # Critical
  frequency           = "PT1M"
  window_size         = "PT1M"

  criteria {
    metric_namespace = "Microsoft.Network/loadBalancers"
    metric_name      = "DipAvailability"
    aggregation      = "Average"
    operator         = "LessThanOrEqual"
    threshold        = 0
  }

  action {
    action_group_id = azurerm_monitor_action_group.observability_critical.id
  }
}
```

> **Health probe configuration:** The Azure LB health probe should target the OTel Collector's `healthcheckextension` endpoint at `HTTP :13133/health`, not just a TCP port check. This validates the collector pipeline is running, not just that the port is open.

### Operator Response

| Condition | Action |
|-----------|--------|
| Availability below 100% (1+ unhealthy) | Check VMSS instance statuses in Azure Portal. Review Activity Log for instance failures. Trigger manual scale-out if VMSS is not auto-recovering. |
| Availability = 0% | All telemetry ingest is halted. Investigate: check VMSS for launch failures (VM quota, image issues), NSG rules, and health probe configuration. |
| Repeated cycling | Instance may be passing health probe then crashing. Check `/var/log/otelcol-contrib` on instances via Azure Bastion. Likely a config issue (bad exporter endpoint, TLS cert error). |

---

## A-05: Backend Storage Disk Saturation

### Problem

Prometheus, Loki, and Tempo each write continuously to Azure Managed Disks. If ingest rate exceeds the estimated 90-day retention sizing, or compaction/retention jobs fail silently, disks fill up. A full disk crashes the backend process.

### Detection Mechanism

Two complementary signals:

1. **Azure Monitor Agent** — `disk_used_percent` metric collected by Azure Monitor Agent (AMA) on each backend VM, published to Azure Monitor Metrics.
2. **`node_exporter` filesystem metrics** — `node_filesystem_avail_bytes / node_filesystem_size_bytes` gives actual utilization percentage, scraped by Prometheus and queryable in Grafana.

### Implementation

#### 1. Azure Monitor Alert — Disk Utilization (via AMA)

Deploy the **Azure Monitor Agent (AMA)** on Prometheus, Loki, and Tempo VMs with a Data Collection Rule that publishes `disk_used_percent`.

```hcl
resource "azurerm_monitor_metric_alert" "disk_warn" {
  for_each = toset(["prometheus", "loki", "tempo"])

  name                = "observability-${each.key}-disk-warning"
  resource_group_name = azurerm_resource_group.observability.name
  scopes              = [azurerm_linux_virtual_machine.backend[each.key].id]
  description         = "WARNING: ${each.key} data volume is >= 80% full — review retention and compaction"
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT5M"

  criteria {
    metric_namespace = "azure.vm.linux.guestmetrics"
    metric_name      = "disk/used_percent"
    aggregation      = "Maximum"
    operator         = "GreaterThanOrEqual"
    threshold        = 80

    dimension {
      name     = "disk"
      operator = "Include"
      values   = ["/data"]
    }
  }

  action {
    action_group_id = azurerm_monitor_action_group.observability_warning.id
  }
}

resource "azurerm_monitor_metric_alert" "disk_critical" {
  for_each = toset(["prometheus", "loki", "tempo"])

  name                = "observability-${each.key}-disk-CRITICAL"
  resource_group_name = azurerm_resource_group.observability.name
  scopes              = [azurerm_linux_virtual_machine.backend[each.key].id]
  description         = "CRITICAL: ${each.key} data volume is >= 90% full — expand disk or data loss will occur"
  severity            = 0
  frequency           = "PT5M"
  window_size         = "PT5M"

  criteria {
    metric_namespace = "azure.vm.linux.guestmetrics"
    metric_name      = "disk/used_percent"
    aggregation      = "Maximum"
    operator         = "GreaterThanOrEqual"
    threshold        = 90

    dimension {
      name     = "disk"
      operator = "Include"
      values   = ["/data"]
    }
  }

  action {
    action_group_id = azurerm_monitor_action_group.observability_critical.id
  }
}
```

#### 2. Grafana Alert Rules (node_exporter — Prometheus datasource)

```yaml
- alert: BackendDiskWarning
  expr: |
    (
      1 - (
        node_filesystem_avail_bytes{job=~"prometheus|loki|tempo", mountpoint="/data"}
        /
        node_filesystem_size_bytes{job=~"prometheus|loki|tempo", mountpoint="/data"}
      )
    ) > 0.80
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "{{ $labels.job }} disk usage exceeds 80% on {{ $labels.instance }}"
    description: "Disk at {{ $value | humanizePercentage }}. Check retention config and compaction status."
    runbook: "https://wiki.company.com/observability/runbooks/backend-disk-saturation"

- alert: BackendDiskCritical
  expr: |
    (
      1 - (
        node_filesystem_avail_bytes{job=~"prometheus|loki|tempo", mountpoint="/data"}
        /
        node_filesystem_size_bytes{job=~"prometheus|loki|tempo", mountpoint="/data"}
      )
    ) > 0.90
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "{{ $labels.job }} disk CRITICAL (>90%) on {{ $labels.instance }} — data loss risk"
    description: "Disk at {{ $value | humanizePercentage }}. Expand Managed Disk or the backend process will crash."
    runbook: "https://wiki.company.com/observability/runbooks/backend-disk-saturation"
```

### Operator Response

| Condition | Action |
|-----------|--------|
| Warning (80–90%) | Verify retention config is correctly set (`2160h`). Confirm compaction is running. If ingest rate has grown, recalculate disk sizing and expand the Managed Disk. |
| Critical (>90%) | **Immediately** expand the Managed Disk via Azure Portal or `az disk update`. For Premium SSD v2, resize is online (no VM restart). Then resize the filesystem (`resize2fs /dev/sdc1`). |
| Disk full / process crashed | Restart the backend after expanding disk. For Prometheus, check for corrupt TSDB blocks. For Loki, validate compactor health. For Tempo, check for orphaned blocks. |

---

## A-06: OTel Gateway VMSS Instance Count Below Floor

### Problem

The Gateway VMSS is configured with a minimum of 2 instances for redundancy. If VMSS repeatedly fails to launch replacement instances (due to quota constraints, VM image issues, or identity errors), the actual instance count falls below the desired minimum.

### Detection Mechanism

Azure Monitor metric on the VMSS resource: compare the current instance count against the configured minimum.

### Implementation

```hcl
resource "azurerm_monitor_metric_alert" "vmss_below_floor" {
  name                = "otel-gateway-vmss-below-minimum"
  resource_group_name = azurerm_resource_group.observability.name
  scopes              = [azurerm_linux_virtual_machine_scale_set.otel_gateway.id]
  description         = "OTel Gateway VMSS has fewer than 2 running instances — capacity is degraded"
  severity            = 0
  frequency           = "PT1M"
  window_size         = "PT5M"

  criteria {
    metric_namespace = "Microsoft.Compute/virtualMachineScaleSets"
    metric_name      = "VirtualMachineScaleSetVMCount"
    aggregation      = "Average"
    operator         = "LessThan"
    threshold        = 2  # matches VMSS minimum capacity
  }

  action {
    action_group_id = azurerm_monitor_action_group.observability_critical.id
  }
}
```

> Window of 5 minutes avoids false alarms during normal rolling upgrades where instances cycle briefly below the floor.

### Operator Response

| Condition | Action |
|-----------|--------|
| Persistent (>5 min) below floor | Check VMSS **Activity Log** for instance creation failures. Common causes: VM quota exhaustion, VM image not found, Managed Identity permission errors. |
| Quota exhaustion | Request quota increase via Azure Portal, or add a second VM SKU family to the VMSS configuration. |
| Correlated with A-04 (LB unhealthy) | Treat as P1 incident — Gateway fleet is degraded and LB is dropping traffic. |

---

## A-07: Prometheus Ingestion Health

### Problem

Prometheus is the sole metrics store. If it stops scraping or its remote-write ingest fails, all metric-based Grafana alerts and dashboards go dark.

### Detection Mechanism

Prometheus exposes self-metrics at `:9090/metrics`:

1. **`up{job="prometheus"}`** — self-scrape target; drops to `0` if the process is down.
2. **`prometheus_remote_storage_failed_samples_total`** — counts samples that failed to write.

### Grafana Alert Rules (Prometheus datasource)

```yaml
- alert: PrometheusDown
  expr: absent(up{job="prometheus"}) or up{job="prometheus"} == 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Prometheus is unreachable or not scraping itself"
    description: "The Prometheus instance has not reported up==1 for 2 minutes. Dashboards and Grafana alerts backed by Prometheus are dark."
    runbook: "https://wiki.company.com/observability/runbooks/prometheus-down"

- alert: PrometheusRemoteWriteErrors
  expr: rate(prometheus_remote_storage_failed_samples_total[5m]) > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Prometheus remote-write is dropping samples"
    description: "{{ $value | humanize }} samples/s are failing to write. Metrics ingest from OTel Gateway is degraded."
    runbook: "https://wiki.company.com/observability/runbooks/prometheus-remote-write-errors"
```

### Operator Response

| Condition | Action |
|-----------|--------|
| `up{job="prometheus"} == 0` | SSH via Azure Bastion to the Prometheus VM. Check process status (`systemctl status prometheus`), inspect logs for TSDB errors or OOM kills. Check disk first (A-05). |
| Remote-write failures | Check OTel Gateway logs for export errors (A-02). Verify Prometheus is accepting remote-write on `:9090/api/v1/write`. Check NSG rules between Gateway and Prometheus. |

---

## A-08: Loki Ingestion Health

### Problem

Loki is the sole log store. If it stops accepting pushes, logs are dropped at the OTel Gateway exporter.

### Detection Mechanism

Loki exposes self-metrics at `:3100/metrics`:

1. **`loki_distributor_lines_received_total`** — rate dropping to zero indicates no logs are being ingested.
2. **`loki_distributor_ingester_appends_failures_total`** — non-zero rate indicates push failures.

### Grafana Alert Rules (Prometheus datasource)

```yaml
- alert: LokiIngestionStalled
  expr: rate(loki_distributor_lines_received_total[10m]) == 0
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "Loki has not received any log lines in 10 minutes"
    description: "Loki distributor line rate is zero. Log ingestion has stopped."
    runbook: "https://wiki.company.com/observability/runbooks/loki-ingestion-stalled"

- alert: LokiIngesterErrors
  expr: rate(loki_distributor_ingester_appends_failures_total[5m]) > 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Loki distributor is recording ingester append failures"
    description: "{{ $value | humanize }} failures/s. Loki ingesters may be overloaded or unhealthy."
    runbook: "https://wiki.company.com/observability/runbooks/loki-ingester-errors"
```

### Operator Response

| Condition | Action |
|-----------|--------|
| Ingestion rate zero | Verify OTel Gateway Loki exporter (A-02 / A-03). SSH via Azure Bastion to Loki VM; check process health and logs. Check disk (A-05). |
| Ingester append failures | Check Loki ingester logs for chunk flush errors or WAL issues. Verify Managed Disk health. If OOM, increase Loki VM memory. |

---

## A-09: Tempo Ingestion Health

### Problem

Tempo is the trace store. If spans stop arriving or Tempo's distributors begin erroring, trace data is silently lost. Tempo's native datasource in Grafana is trace-search only — it cannot back alert rules.

### Detection Mechanism

Tempo exposes self-metrics at `:3200/metrics`:

1. **`tempo_distributor_spans_received_total`** — rate dropping to zero means no spans are being received.
2. **`tempo_request_duration_seconds_count{status_code=~"5.."}`** — ingester or querier error rate.

### Grafana Alert Rules (Prometheus datasource — NOT Tempo datasource)

```yaml
- alert: TempoIngestionStalled
  expr: rate(tempo_distributor_spans_received_total[10m]) == 0
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Tempo has not received any spans in 10 minutes"
    description: "Tempo distributor span rate is zero. Trace ingestion has stopped."
    runbook: "https://wiki.company.com/observability/runbooks/tempo-ingestion-stalled"

- alert: TempoIngesterErrors
  expr: rate(tempo_request_duration_seconds_count{status_code=~"5.."}[5m]) > 0.1
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Tempo is returning 5xx errors on ingester/querier requests"
    description: "{{ $value | humanize }} error req/s. Tempo ingesters or querier may be degraded."
    runbook: "https://wiki.company.com/observability/runbooks/tempo-ingester-errors"
```

### Operator Response

| Condition | Action |
|-----------|--------|
| Span ingestion zero (warning) | Verify OTel Gateway OTLP exporter (A-02). Check Tempo distributor logs. Confirm instrumented services are running. |
| 5xx error rate on ingester | SSH via Azure Bastion to Tempo VM; inspect ingester logs. Check WAL Managed Disk health. Verify block flushing to Blob Storage is not blocked (Managed Identity permissions, Blob container access). |

---

## Platform Self-Monitoring Summary

| Component | Alert | Mechanism |
|-----------|-------|-----------|
| OTel Gateway — export failures | [A-02](#a-02-otel-gateway-export-errors) | Prometheus metrics → Grafana alert |
| OTel Gateway — queue saturation | [A-03](#a-03-otel-gateway-queue-saturation) | Prometheus metrics → Grafana alert |
| Azure LB — unhealthy backend pool | [A-04](#a-04-azure-lb-backend-pool-unhealthy) | Azure Monitor Alert → Action Group |
| Prometheus / Loki / Tempo — disk full | [A-05](#a-05-backend-storage-disk-saturation) | AMA + node_exporter → Azure Monitor Alert + Grafana alert |
| Gateway VMSS — below minimum capacity | [A-06](#a-06-otel-gateway-vmss-instance-count-below-floor) | Azure Monitor Alert → Action Group |
| ACA sidecar — telemetry disruption | [A-01](#a-01-aca-sidecar-telemetry-disruption) | Azure Monitor (Container App diagnostics) → Action Group |
| Prometheus — process down / remote-write errors | [A-07](#a-07-prometheus-ingestion-health) | Prometheus self-metrics (via Prometheus DS) → Grafana alert |
| Loki — ingestion stalled / ingester errors | [A-08](#a-08-loki-ingestion-health) | Prometheus / Loki self-metrics → Grafana alert |
| Tempo — span ingestion stalled / 5xx errors | [A-09](#a-09-tempo-ingestion-health) | Prometheus (Tempo self-metrics, not Tempo DS) → Grafana alert |

### OTel Platform Health Dashboard — Panel Inventory

| Panel | Query / Source | Alert linked |
|-------|---------------|--------------|
| Gateway export error rate (by exporter) | `rate(otelcol_exporter_send_failed_*[5m])` — Prometheus | A-02 |
| Gateway enqueue failure rate | `rate(otelcol_exporter_enqueue_failed_*[5m])` — Prometheus | A-02 |
| Gateway queue fill % (by exporter) | `otelcol_exporter_queue_size / otelcol_exporter_queue_capacity` — Prometheus | A-03 |
| Gateway spans/metrics/logs received (by receiver) | `rate(otelcol_receiver_accepted_*[5m])` — Prometheus | — |
| Gateway spans/metrics/logs exported (by exporter) | `rate(otelcol_exporter_sent_*[5m])` — Prometheus | — |
| Gateway CPU utilization (VMSS instances) | `node_cpu_seconds_total` — Prometheus | — |
| Azure LB data path availability | `DipAvailability` — Azure Monitor datasource | A-04 |
| Azure LB health probe status | `HealthProbeStatus` — Azure Monitor datasource | A-04 |
| Prometheus disk usage % | `node_filesystem_avail_bytes / node_filesystem_size_bytes` — Prometheus | A-05 |
| Loki disk usage % | same metric, `job="loki"` | A-05 |
| Tempo disk usage % | same metric, `job="tempo"` | A-05 |
| VMSS instance count | `VirtualMachineScaleSetVMCount` — Azure Monitor datasource | A-06 |
| ACA sidecar restart count (last 1h) | `RestartCount` — Azure Monitor datasource | A-01 |
| Prometheus up / remote-write error rate | `up{job="prometheus"}`, `rate(prometheus_remote_storage_failed_samples_total[5m])` — Prometheus | A-07 |
| Loki distributor line rate / ingester error rate | `rate(loki_distributor_lines_received_total[5m])`, `rate(loki_distributor_ingester_appends_failures_total[5m])` — Prometheus | A-08 |
| Tempo distributor span rate / 5xx error rate | `rate(tempo_distributor_spans_received_total[5m])`, `rate(tempo_request_duration_seconds_count{status_code=~"5.."}[5m])` — Prometheus | A-09 |

### Action Groups

All alerts route to one of two Action Groups:

| Action Group | Severity | Receivers |
|-------------|----------|-----------|
| `observability-alerts-critical` | Critical | PagerDuty (webhook), Slack `#observability-alerts` (webhook), on-call email DL |
| `observability-alerts-warning` | Warning | Slack `#observability-alerts` (webhook), on-call email DL |

### Related Items

- **presentation.md** — Managed Disk Sizing Note cross-references A-05 for disk saturation alerting
- **futures.md** — F1 (persistent queue: telemetry loss during sidecar restart window is a known gap)
