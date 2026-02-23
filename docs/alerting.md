# Alerting Strategy

Alerting for the Brighton Collectibles observability platform is implemented at multiple layers: **AWS-native** (EventBridge + CloudWatch Alarms) for infrastructure and container lifecycle events, and **Grafana Managed Alerting** (backed by Prometheus, Loki query, or CloudWatch datasource) for signal-level thresholds and platform health.

> **Backend health coverage (A-07 / A-08 / A-09):** Prometheus, Loki, and Tempo each expose Prometheus-format self-metrics (ingestion counters, error rates, internal health). AMG alert rules query these via the **Prometheus datasource** (PromQL). The Loki datasource also supports LogQL metric queries for alerting. The **Tempo datasource cannot back AMG alert rules** — Tempo health alerts use the Prometheus datasource scraping Tempo's `/metrics` endpoint instead.

For a visual overview of how both paths deliver notifications to operators, see [alert_diagram.md](alert_diagram.md).

This document records each discrete alerting mechanism: what it detects, how it is implemented, and expected operator response.

---

## Index

| ID | Alert | Severity | Signal Source | Status |
|----|-------|----------|---------------|--------|
| [A-01](#a-01-ecs-sidecar-silent-telemetry-blackout) | ECS Sidecar silent telemetry blackout | **Critical** | EventBridge (ECS task state change) | Active |
| [A-02](#a-02-otel-gateway-export-errors) | OTel Gateway export errors | **Critical** | Prometheus (otelcol internal metrics) | Active |
| [A-03](#a-03-otel-gateway-queue-saturation) | OTel Gateway queue saturation | **Warning → Critical** | Prometheus (otelcol internal metrics) | Active |
| [A-04](#a-04-nlb-target-group-unhealthy-hosts) | NLB target group unhealthy hosts | **Critical** | CloudWatch (NLB target health) | Active |
| [A-05](#a-05-backend-storage-disk-saturation) | Backend storage disk saturation (Prometheus / Loki / Tempo) | **Warning → Critical** | CloudWatch (EBS utilization) + Prometheus (node_exporter) | Active |
| [A-06](#a-06-otel-gateway-asg-instance-count-below-floor) | OTel Gateway ASG instance count below floor | **Critical** | CloudWatch (EC2 Auto Scaling) | Active |
| [A-07](#a-07-prometheus-ingestion-health) | Prometheus ingestion health | **Critical** | Prometheus (self-metrics via Prometheus DS on AMG) | Active |
| [A-08](#a-08-loki-ingestion-health) | Loki ingestion health | **Critical** | Prometheus (Loki self-metrics via Prometheus DS on AMG) | Active |
| [A-09](#a-09-tempo-ingestion-health) | Tempo ingestion health | **Warning → Critical** | Prometheus (Tempo self-metrics via Prometheus DS on AMG) | Active |

---

## A-01: ECS Sidecar Silent Telemetry Blackout

### Problem

The `otelcol-contrib` sidecar container in every ECS task is marked `essential: false` so that a collector crash does not take down the application. The trade-off is that when the sidecar stops, the task continues running and emitting no telemetry — traces, metrics, and logs for that task are silently lost until the sidecar is restarted or the task is replaced.

Because nothing breaks from the application's point of view, this condition can persist undetected indefinitely.

### Detection Mechanism

Amazon ECS publishes **ECS Task State Change** events to Amazon EventBridge whenever a container within a task changes state. When the `otelcol-contrib` container exits (crash, OOM, or misconfiguration), ECS emits an event where:

- `detail.lastStatus` = `RUNNING` — the task is still alive (sidecar is non-essential)
- `detail.containers[*]` contains an entry where `name` = `otelcol-contrib` and `lastStatus` = `STOPPED`

This event pattern unambiguously identifies a task with a dead sidecar and a live app container.

### Implementation

#### 1. EventBridge Rule

Create an EventBridge rule in each AWS region where ECS clusters are deployed.

**Event pattern:**

```json
{
  "source": ["aws.ecs"],
  "detail-type": ["ECS Task State Change"],
  "detail": {
    "lastStatus": ["RUNNING"],
    "containers": {
      "name": ["otelcol-contrib"],
      "lastStatus": ["STOPPED"]
    }
  }
}
```

> **Note:** EventBridge content filtering on array-of-objects fields uses per-element matching. The pattern above matches any task state change event that includes at least one container named `otelcol-contrib` with `lastStatus: STOPPED`, while the task itself remains `RUNNING`.

**Terraform resource (reference):**

```hcl
resource "aws_cloudwatch_event_rule" "ecs_sidecar_stopped" {
  name        = "ecs-otelcol-sidecar-stopped"
  description = "Fires when otelcol-contrib sidecar stops but ECS task is still running"

  event_pattern = jsonencode({
    source        = ["aws.ecs"]
    "detail-type" = ["ECS Task State Change"]
    detail = {
      lastStatus = ["RUNNING"]
      containers = {
        name       = ["otelcol-contrib"]
        lastStatus = ["STOPPED"]
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "ecs_sidecar_stopped_sns" {
  rule      = aws_cloudwatch_event_rule.ecs_sidecar_stopped.name
  target_id = "SendToSNS"
  arn       = aws_sns_topic.observability_alerts_critical.arn

  input_transformer {
    input_paths = {
      cluster   = "$.detail.clusterArn"
      task      = "$.detail.taskArn"
      service   = "$.detail.group"
      region    = "$.region"
      exit_code = "$.detail.containers[?(@.name=='otelcol-contrib')].exitCode"
    }
    input_template = <<-EOT
      "CRITICAL: OTel sidecar stopped on a running ECS task — telemetry collection suspended.
      Cluster: <cluster>
      Task:    <task>
      Service: <service>
      Region:  <region>
      Sidecar exit code: <exit_code>
      
      Action: Stop and replace the task to restart the sidecar, or investigate the exit code.
      Runbook: https://wiki.company.com/observability/runbooks/ecs-sidecar-stopped"
    EOT
  }
}
```

#### 2. SNS Topic → Notification Channels

Route the SNS topic (`observability_alerts_critical`) to:

| Channel | Integration |
|---------|-------------|
| PagerDuty | SNS → PagerDuty Events API v2 subscription (Critical severity) |
| Slack `#observability-alerts` | SNS → Lambda → Slack Incoming Webhook |
| Email (on-call DL) | SNS email subscription as fallback |

#### 3. CloudWatch Metric Filter + Alarm (secondary signal)

As a second layer, create a CloudWatch metric filter on the sidecar's own log group (`/ecs/otelcol-contrib`) to count fatal exit events and alarm when they exceed 0 in a rolling 5-minute window.

```hcl
resource "aws_cloudwatch_log_metric_filter" "sidecar_fatal" {
  name           = "otelcol-sidecar-fatal-exit"
  log_group_name = "/ecs/otelcol-contrib"
  pattern        = "?\"Fatal error\" ?\"failed to start\" ?\"exiting with error\""

  metric_transformation {
    name      = "OtelSidecarFatalExit"
    namespace = "Brighton/Observability"
    value     = "1"
    default_value = "0"
  }
}

resource "aws_cloudwatch_metric_alarm" "sidecar_fatal_alarm" {
  alarm_name          = "ecs-otelcol-sidecar-fatal-exit"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "OtelSidecarFatalExit"
  namespace           = "Brighton/Observability"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "otelcol-contrib sidecar emitted a fatal log line in the last 5 minutes"
  alarm_actions       = [aws_sns_topic.observability_alerts_critical.arn]
  treat_missing_data  = "notBreaching"
}
```

#### 4. Grafana Alert (platform health dashboard)

Add a panel to the **OTel Platform Health** dashboard that queries the CloudWatch metric via the AMG CloudWatch datasource, with a Grafana alert rule configured to fire when `OtelSidecarFatalExit` sum > 0 for 5 minutes.

This surfaces the condition in Grafana alongside Gateway queue depth and backend health panels, giving operators a single-pane incident view.

```
Alert rule name : ECS Sidecar Fatal Exit
Datasource      : CloudWatch (AMG)
Namespace       : Brighton/Observability
Metric          : OtelSidecarFatalExit
Stat            : Sum
Period          : 5m
Condition       : IS ABOVE 0
For             : 5m
Severity label  : critical
```

---

### Operator Response

| Condition | Action |
|-----------|--------|
| Single task, non-repeating | Trigger a new task deployment: `aws ecs stop-task --task <arn>` — ECS will launch a replacement with a fresh sidecar. |
| Multiple tasks, same cluster | Check Gateway NLB target health and sidecar config validity. A bad config (invalid YAML, wrong endpoint) will cause all sidecars to fail at startup. Roll back the config source (S3/SSM). |
| Exit code `137` (OOM) | Increase sidecar memory reservation in the task definition (from 128 MiB → 256 MiB). Review batch size and memory limiter thresholds in the sidecar collector config. |
| Exit code `1` / startup failure | Pull sidecar logs from CloudWatch (`/ecs/otelcol-contrib`) and inspect the startup error. Common causes: invalid YAML, unreachable Gateway endpoint, TLS cert mismatch. |
| Recurring across rolling deploys | Investigate whether the ECS task execution role has `GetParameter` / `s3:GetObject` permissions to fetch the collector config. |

---

### Related Items

- **endpoints.md** — [ECS Sidecar Container Requirements](#ecs-fargate--ec2-sidecar-otel-agent) — `essential: false` deployment note
- **arch_eval.md** — P1 #11 (resolved)
- **arch_eval.md** — P1 #12 (platform self-monitoring — see A-02 through A-06 below)
- **arch_eval.md** — P3 #25 (full alerting strategy — see all alerts in this document)
- **futures.md** — F1 (persistent queue: telemetry loss during sidecar restart window is a known gap)

---

## A-02: OTel Gateway Export Errors

### Problem

The OTel Gateway Collectors export processed telemetry to Prometheus (remote-write), Loki, and Tempo. If any exporter begins failing — due to a backend being down, a misconfigured endpoint, authentication failure, or network disruption — telemetry is silently dropped after the retry budget is exhausted. Nothing notifies the operator.

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

Prometheus scrapes these metrics from each Gateway instance via the ASG's target discovery. AMG Grafana evaluates alert rules directly against the Prometheus datasource.

### Implementation

#### 1. Prometheus Scrape Config (Gateway instances)

Add a scrape job to the Prometheus server for the Gateway ASG:

```yaml
scrape_configs:
  - job_name: otelcol_gateway
    ec2_sd_configs:
      - region: us-east-1
        port: 8888
        filters:
          - name: tag:Role
            values: [otel-gateway]
    relabel_configs:
      - source_labels: [__meta_ec2_instance_id]
        target_label: instance
      - source_labels: [__meta_ec2_tag_Environment]
        target_label: environment
```

#### 2. Grafana Alert Rules

```yaml
# Alert fires when any exporter drops telemetry for 5 continuous minutes

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

Route fired alerts through Grafana's notification policy → **contact point: SNS** → PagerDuty / Slack `#observability-alerts` / email (same SNS topic used by A-01).

### Operator Response

| Condition | Action |
|-----------|--------|
| Export failures for one exporter | Check the specific backend (Prometheus / Loki / Tempo): `systemctl status prometheus` or EBS volume saturation alarms. |
| All exporters failing simultaneously | Verify Gateway instance connectivity to backends. Check VPC security groups; confirm target hosts are running. |
| Enqueue failures | Backend is not recovering fast enough. Scale the backend vertically, or temporarily reduce ingest rate at the agent level by increasing batch intervals. |
| After recovery | Confirm failure counter reset; verify telemetry resumed in Grafana Explorer for each signal. |

---

## A-03: OTel Gateway Queue Saturation

### Problem

The OTel Gateway uses an in-memory queue (`queue_size`) to buffer telemetry before export. If a backend is slow or down, the queue fills up. Once at capacity, incoming telemetry is dropped. An operator should be alerted early — when the queue is filling but before it reaches capacity — so action can be taken before data loss begins.

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
| Warning (70–90%) | Investigate backend health (see A-02 response). Check if an ASG scale-out event is needed. |
| Critical (>90%) | Immediate backend investigation. Consider increasing `queue_size` in the Gateway config as a temporary measure; this delays drops but does not fix the root cause. |
| Queue full + export errors (A-02 also firing) | Backend is down or unreachable; escalate to on-call. Data loss is occurring. |

---

## A-04: NLB Target Group Unhealthy Hosts

### Problem

The NLB fronts the OTel Gateway ASG. If all Gateway instances become unhealthy, inbound OTLP traffic from 1000+ collector agents is silently discarded at the NLB — no back-pressure, no error returned to the push source (UDP-style drop). The NLB `HealthyHostCount` metric in CloudWatch is the authoritative signal.

### Detection Mechanism

CloudWatch metric `HealthyHostCount` on the NLB target group. The ASG minimum is 2 instances; a `HealthyHostCount` below this floor indicates degraded capacity. A count of 0 is a full platform outage.

### Implementation

```hcl
resource "aws_cloudwatch_metric_alarm" "nlb_unhealthy_hosts_warning" {
  alarm_name          = "nlb-otel-gateway-unhealthy-hosts-warning"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HealthyHostCount"
  namespace           = "AWS/NetworkELB"
  period              = 60
  statistic           = "Minimum"
  threshold           = 2   # ASG minimum capacity; adjust if ASG floor changes
  alarm_description   = "NLB target group has fewer than 2 healthy Gateway hosts — capacity is degraded"
  alarm_actions       = [aws_sns_topic.observability_alerts_warning.arn]
  ok_actions          = [aws_sns_topic.observability_alerts_warning.arn]
  treat_missing_data  = "breaching"

  dimensions = {
    LoadBalancer = aws_lb.otel_nlb.arn_suffix
    TargetGroup  = aws_lb_target_group.otel_gateway.arn_suffix
  }
}

resource "aws_cloudwatch_metric_alarm" "nlb_no_healthy_hosts" {
  alarm_name          = "nlb-otel-gateway-no-healthy-hosts-CRITICAL"
  comparison_operator = "LessThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "HealthyHostCount"
  namespace           = "AWS/NetworkELB"
  period              = 60
  statistic           = "Minimum"
  threshold           = 0
  alarm_description   = "CRITICAL: NLB has zero healthy Gateway hosts — all inbound telemetry is being dropped"
  alarm_actions       = [aws_sns_topic.observability_alerts_critical.arn]
  ok_actions          = [aws_sns_topic.observability_alerts_critical.arn]
  treat_missing_data  = "breaching"

  dimensions = {
    LoadBalancer = aws_lb.otel_nlb.arn_suffix
    TargetGroup  = aws_lb_target_group.otel_gateway.arn_suffix
  }
}
```

> **Note:** `treat_missing_data = "breaching"` ensures that if CloudWatch stops receiving the metric (e.g., because all instances have terminated), the alarm immediately enters ALARM state rather than waiting for the evaluation period.

### Operator Response

| Condition | Action |
|-----------|--------|
| Count below floor (1 healthy host) | Verify remaining instance health via EC2 console. Check ASG activity history for launch failures. Trigger a manual scale-out if ASG is not auto-recovering. |
| Count = 0 | All telemetry ingest is halted. Immediately investigate: check ASG launch failures (instance type availability, AMI issues), security group rules, and health check configuration. |
| Repeated cycling | Instance may be passing health check then crashing. Check `/var/log/otelcol-contrib` on instances. Likely a config issue (bad exporter endpoint, TLS cert error). |

---

## A-05: Backend Storage Disk Saturation

### Problem

Prometheus, Loki, and Tempo each write continuously to EBS volumes. If ingest rate exceeds the estimated 90-day retention sizing, or compaction/retention jobs fail silently, volumes fill up. A full disk crashes the backend process — Prometheus stops ingesting, Loki rejects log pushes, Tempo drops traces. No operator is notified unless an alert is configured.

### Detection Mechanism

Two complementary signals:

1. **CloudWatch EBS `VolumeConsumedReadWriteOps` and `VolumeIdleTime`** — detects I/O saturation (disk busy but not necessarily full).
2. **`node_exporter` filesystem metrics** — `node_filesystem_avail_bytes / node_filesystem_size_bytes` gives actual utilization percentage, scraped by Prometheus and queryable in Grafana.

The `node_exporter` signal is more actionable (actual free space); the CloudWatch signal is a secondary indicator of I/O pressure.

### Implementation

#### 1. CloudWatch Alarm — EBS Utilization (via `df`/CW Agent)

Deploy the **CloudWatch Agent** on Prometheus, Loki, and Tempo EC2 instances to publish `disk_used_percent` from CW Agent's disk plugin. (Alternatively, use `node_exporter` alone if Prometheus already scrapes these hosts.)

```hcl
resource "aws_cloudwatch_metric_alarm" "disk_warn" {
  for_each = toset(["prometheus", "loki", "tempo"])

  alarm_name          = "observability-${each.key}-disk-warning"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "disk_used_percent"
  namespace           = "CWAgent"
  period              = 300
  statistic           = "Maximum"
  threshold           = 80
  alarm_description   = "WARNING: ${each.key} data volume is >= 80% full — review retention and compaction"
  alarm_actions       = [aws_sns_topic.observability_alerts_warning.arn]
  ok_actions          = [aws_sns_topic.observability_alerts_warning.arn]
  treat_missing_data  = "breaching"

  dimensions = {
    InstanceId = var.backend_instance_ids[each.key]
    path       = "/data"   # mount point for TSDB / chunks / blocks
    fstype     = "ext4"
  }
}

resource "aws_cloudwatch_metric_alarm" "disk_critical" {
  for_each = toset(["prometheus", "loki", "tempo"])

  alarm_name          = "observability-${each.key}-disk-CRITICAL"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "disk_used_percent"
  namespace           = "CWAgent"
  period              = 300
  statistic           = "Maximum"
  threshold           = 90
  alarm_description   = "CRITICAL: ${each.key} data volume is >= 90% full — expand EBS or data loss will occur"
  alarm_actions       = [aws_sns_topic.observability_alerts_critical.arn]
  ok_actions          = [aws_sns_topic.observability_alerts_critical.arn]
  treat_missing_data  = "breaching"

  dimensions = {
    InstanceId = var.backend_instance_ids[each.key]
    path       = "/data"
    fstype     = "ext4"
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
    description: "Disk at {{ $value | humanizePercentage }}. Expand EBS volume or the backend process will crash."
    runbook: "https://wiki.company.com/observability/runbooks/backend-disk-saturation"
```

### Operator Response

| Condition | Action |
|-----------|--------|
| Warning (80–90%) | Verify retention config is correctly set (`2160h`). Confirm compaction is running (Loki compactor logs, Tempo compactor logs, Prometheus TSDB compaction stats). If ingest rate has grown, recalculate EBS sizing and expand volume. |
| Critical (>90%) | **Immediately** expand the EBS volume via AWS Console (`ModifyVolume`) and resize the filesystem (`resize2fs /dev/nvme1n1`). No instance restart required for gp3. |
| Disk full / process crashed | Restart the backend after expanding disk. For Prometheus, check for corrupt TSDB blocks and run `promtool tsdb analyze`. For Loki, validate the compactor is healthy. For Tempo, check for orphaned blocks. |

---

## A-06: OTel Gateway ASG Instance Count Below Floor

### Problem

The Gateway ASG is configured with a minimum of 2 instances for redundancy. If Auto Scaling repeatedly fails to launch replacement instances (due to capacity constraints, launch template errors, or IAM issues), the ASG GroupDesiredCapacity remains correct but GroupInServiceInstances falls below the desired count. This is distinct from the NLB health check (A-04) because an instance can be in-service from the ASG perspective but still unhealthy at the NLB layer.

### Detection Mechanism

CloudWatch metric `GroupInServiceInstances` in the `AWS/AutoScaling` namespace.

### Implementation

```hcl
resource "aws_cloudwatch_metric_alarm" "asg_below_floor" {
  alarm_name          = "otel-gateway-asg-below-minimum"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 3
  metric_name         = "GroupInServiceInstances"
  namespace           = "AWS/AutoScaling"
  period              = 60
  statistic           = "Minimum"
  threshold           = 2   # matches ASG min_size
  alarm_description   = "OTel Gateway ASG has fewer than 2 in-service instances — capacity is degraded"
  alarm_actions       = [aws_sns_topic.observability_alerts_critical.arn]
  ok_actions          = [aws_sns_topic.observability_alerts_critical.arn]
  treat_missing_data  = "breaching"

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.otel_gateway.name
  }
}
```

> Set `evaluation_periods = 3` (3 × 60 s = 3 minutes) to avoid false alarms during normal rolling deployments, where instances cycle briefly below the floor.

### Operator Response

| Condition | Action |
|-----------|--------|
| Persistent (>3 min) below floor | Check ASG **Activity History** for launch failure reasons. Common causes: EC2 capacity constraints (try alternate AZ or instance type), launch template referencing a deleted AMI, or instance profile permission error. |
| Launch failures due to capacity | Add a second instance type to the ASG launch template (e.g., `t3.medium` + `t3a.medium`) using a mixed instances policy. |
| Correlated with A-04 (NLB unhealthy) | Treat as P1 incident — Gateway fleet is degraded and NLB is routing to unhealthy targets or dropping traffic. |

---

## A-07: Prometheus Ingestion Health

### Problem

Prometheus is the sole metrics store. If it stops scraping or its remote-write ingest fails — due to TSDB corruption, OOM, or a storage issue — all metric-based Grafana alerts and dashboards go dark. The failure may be invisible until operators notice stale dashboards.

### Detection Mechanism

Prometheus exposes Prometheus-format self-metrics at `:9090/metrics`. Two complementary signals:

1. **`up{job="prometheus"}`** — Prometheus scrapes itself; if this drops to `0` the process is down or unreachable.
2. **`prometheus_remote_storage_failed_samples_total`** — counts samples that failed to write to remote-write endpoint (OTel Gateway → Prometheus). A non-zero rate indicates ingest failures.

### AMG Compatibility

Both queries use **PromQL against the Prometheus datasource** in AMG. This is fully supported for Grafana Managed Alerting rules. ✓

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
| `up{job="prometheus"} == 0` | SSH to the Prometheus EC2 instance. Check process status (`systemctl status prometheus`), inspect logs for TSDB errors or OOM kills. Restart if safe; check disk first (see A-05). |
| Remote-write failures | Check OTel Gateway logs for export errors (see A-02). Verify Prometheus is accepting remote-write on `:9090/api/v1/write`. Check network path from Gateway to Prometheus (security group, VPC routing). |

---

## A-08: Loki Ingestion Health

### Problem

Loki is the sole log store. If it stops accepting pushes — due to ingester OOM, distributor errors, or a storage issue — logs are dropped at the OTel Gateway exporter. The OTel Gateway will report exporter errors (A-02), but a Loki-specific alert provides a targeted signal for faster diagnosis.

### Detection Mechanism

Loki exposes Prometheus-format self-metrics at `:3100/metrics`. Two signals:

1. **`loki_distributor_lines_received_total`** — rate dropping to zero indicates no logs are being ingested (use `absent()` or a rate-drops-to-zero condition).
2. **`loki_request_duration_seconds_count{status_code=~"5.."}` / `loki_distributor_ingester_appends_failures_total`** — non-zero rate indicates push failures or ingester errors.

### AMG Compatibility

Both queries use **PromQL against the Prometheus datasource** in AMG (Prometheus must scrape Loki's `:3100/metrics` endpoint). Alternatively, LogQL metric queries against the **Loki datasource** can back AMG alert rules for log-volume conditions. ✓ Both paths supported.

### Grafana Alert Rules (Prometheus datasource)

```yaml
- alert: LokiIngestionStalled
  expr: rate(loki_distributor_lines_received_total[10m]) == 0
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "Loki has not received any log lines in 10 minutes"
    description: "Loki distributor line rate is zero. Log ingestion has stopped — OTel Gateway Loki exporter may be failing."
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
| Ingestion rate zero | Verify OTel Gateway Loki exporter (check A-02 / A-03 for queue backup). SSH to Loki host and check process health (`systemctl status loki`) and ingester logs. Check disk (A-05). |
| Ingester append failures | Check Loki ingester logs for chunk flush errors or WAL issues. Verify EBS WAL volume health. If OOM, increase Loki ingester memory limits. |

---

## A-09: Tempo Ingestion Health

### Problem

Tempo is the trace store. If spans stop arriving or Tempo's distributors begin erroring, trace data is silently lost. Tempo's native datasource in Grafana is trace-search only — it cannot back alert rules. Health monitoring must use Tempo's Prometheus-format self-metrics scraped by Prometheus and queried via the Prometheus datasource.

### Detection Mechanism

Tempo exposes Prometheus-format self-metrics at `:3200/metrics`. Two signals:

1. **`tempo_distributor_spans_received_total`** — rate dropping to zero means no spans are being received.
2. **`tempo_request_error_rate`** (or `rate(tempo_request_duration_seconds_count{status_code=~"5.."}[5m])`) — ingester or querier error rate.

### AMG Compatibility

Queries use **PromQL against the Prometheus datasource** in AMG. Prometheus must scrape Tempo's `:3200/metrics` endpoint. The **Tempo datasource cannot be used for AMG alert rules** — Grafana's Tempo datasource is trace-query only with no alerting support. ✓ Prometheus datasource path is fully supported.

### Grafana Alert Rules (Prometheus datasource — NOT Tempo datasource)

```yaml
- alert: TempoIngestionStalled
  expr: rate(tempo_distributor_spans_received_total[10m]) == 0
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Tempo has not received any spans in 10 minutes"
    description: "Tempo distributor span rate is zero. Trace ingestion has stopped — check OTel Gateway OTLP exporter (A-02) and application instrumentation coverage."
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

> **Note on zero-span alerting:** A rate-drops-to-zero condition for Tempo should be treated as **Warning**, not Critical. Trace instrumentation coverage is partial — some services may not emit spans. Validate against expected instrumented services before treating this as a hard failure.

### Operator Response

| Condition | Action |
|-----------|--------|
| Span ingestion zero (warning) | Verify OTel Gateway OTLP exporter to Tempo (A-02). Check Tempo distributor logs on `:3200`. Confirm instrumented services are running. |
| 5xx error rate on ingester | SSH to Tempo host; inspect ingester logs. Check WAL EBS health. Verify block flushing to S3 is not blocked (IAM role, S3 bucket policy). |
| Query errors (Tempo datasource in Grafana) | Check Tempo querier logs. Verify S3 read permissions. Run `tempo-cli analyse blocks` if blocks appear missing. |

---

## Platform Self-Monitoring Summary

The following table maps each platform component to its alert coverage:

| Component | Alert | Mechanism |
|-----------|-------|-----------|
| OTel Gateway — export failures | [A-02](#a-02-otel-gateway-export-errors) | Prometheus metrics → Grafana alert |
| OTel Gateway — queue saturation | [A-03](#a-03-otel-gateway-queue-saturation) | Prometheus metrics → Grafana alert |
| NLB — unhealthy target hosts | [A-04](#a-04-nlb-target-group-unhealthy-hosts) | CloudWatch Alarm → SNS |
| Prometheus / Loki / Tempo — disk full | [A-05](#a-05-backend-storage-disk-saturation) | CW Agent + node_exporter → CW Alarm + Grafana alert |
| Gateway ASG — below minimum capacity | [A-06](#a-06-otel-gateway-asg-instance-count-below-floor) | CloudWatch Alarm → SNS |
| ECS sidecar — silent blackout | [A-01](#a-01-ecs-sidecar-silent-telemetry-blackout) | EventBridge → SNS |
| Prometheus — process down / remote-write errors | [A-07](#a-07-prometheus-ingestion-health) | Prometheus self-metrics (via Prometheus DS) → Grafana alert |
| Loki — ingestion stalled / ingester errors | [A-08](#a-08-loki-ingestion-health) | Prometheus / Loki self-metrics → Grafana alert |
| Tempo — span ingestion stalled / 5xx errors | [A-09](#a-09-tempo-ingestion-health) | Prometheus (Tempo self-metrics, not Tempo DS) → Grafana alert |

### OTel Platform Health Dashboard — Panel Inventory

The following panels should be present on the **OTel Platform Health** dashboard in AWS Managed Grafana. This dashboard is the single-pane view for the self-monitoring posture of the observability stack.

| Panel | Query / Source | Alert linked |
|-------|---------------|--------------|
| Gateway export error rate (by exporter) | `rate(otelcol_exporter_send_failed_*[5m])` — Prometheus | A-02 |
| Gateway enqueue failure rate | `rate(otelcol_exporter_enqueue_failed_*[5m])` — Prometheus | A-02 |
| Gateway queue fill % (by exporter) | `otelcol_exporter_queue_size / otelcol_exporter_queue_capacity` — Prometheus | A-03 |
| Gateway spans/metrics/logs received (by receiver) | `rate(otelcol_receiver_accepted_*[5m])` — Prometheus | — |
| Gateway spans/metrics/logs exported (by exporter) | `rate(otelcol_exporter_sent_*[5m])` — Prometheus | — |
| Gateway CPU utilization (ASG instances) | `node_cpu_seconds_total` — Prometheus | — |
| NLB healthy host count | `HealthyHostCount` — CloudWatch datasource | A-04 |
| NLB new flow count and processed bytes | `NewFlowCount`, `ProcessedBytes` — CloudWatch datasource | — |
| Prometheus disk usage % | `node_filesystem_avail_bytes / node_filesystem_size_bytes` — Prometheus | A-05 |
| Loki disk usage % | same metric, `job="loki"` | A-05 |
| Tempo disk usage % | same metric, `job="tempo"` | A-05 |
| ASG in-service instance count | `GroupInServiceInstances` — CloudWatch datasource | A-06 |
| ECS sidecar fatal exits (last 1h) | `OtelSidecarFatalExit` — CloudWatch datasource | A-01 |
| Prometheus up / remote-write error rate | `up{job="prometheus"}`, `rate(prometheus_remote_storage_failed_samples_total[5m])` — Prometheus | A-07 |
| Loki distributor line rate / ingester error rate | `rate(loki_distributor_lines_received_total[5m])`, `rate(loki_distributor_ingester_appends_failures_total[5m])` — Prometheus | A-08 |
| Tempo distributor span rate / 5xx error rate | `rate(tempo_distributor_spans_received_total[5m])`, `rate(tempo_request_duration_seconds_count{status_code=~"5.."}[5m])` — Prometheus | A-09 |

### SNS Topics

All alerts route to one of two SNS topics:

| Topic | Severity | Subscribers |
|-------|----------|-------------|
| `observability_alerts_critical` | Critical | PagerDuty (P1), Slack `#observability-alerts`, on-call email DL |
| `observability_alerts_warning` | Warning | Slack `#observability-alerts`, on-call email DL |

### Related Items

- **arch_eval.md** — P1 #12 (resolved)
- **arch_eval.md** — P3 #25 (alerting strategy — this document is the authoritative record)
- **presentation.md** — EBS Sizing Note cross-references A-05 for disk saturation alerting
- **futures.md** — F1 (persistent queue), F2 (Gateway detailed config)
