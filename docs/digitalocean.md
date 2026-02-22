# DigitalOcean — Observability Integration

## Overview

DigitalOcean (DO) is a cloud infrastructure provider hosting Brighton Collectibles' **Python scheduled jobs**. These jobs run on **Droplets** (Linux VMs), making the integration pattern identical to on-prem Linux hosts: an `otelcol-contrib` agent runs as a `systemd` service on each Droplet, collecting host metrics, application logs, and traces, then forwarding all telemetry via TLS OTLP over a WireGuard VPN tunnel to the Gateway NLB at `nlb.company.com:443`.

---

## Telemetry Coverage

### Metrics

| Metric | Source | Purpose |
|---|---|---|
| `job.duration.seconds` | OTel Python SDK | SLA compliance; detect slow jobs |
| `job.runs.success.total` | OTel Python SDK | Job reliability tracking |
| `job.runs.failure.total` | OTel Python SDK | Alert on failures |
| `job.records.processed.total` | OTel Python SDK | ETL/pipeline throughput |
| `system.cpu.utilization` | `hostmetrics` receiver | Right-sizing; runaway job detection |
| `system.memory.utilization` | `hostmetrics` receiver | Memory leak detection |
| `system.disk.io` | `hostmetrics` receiver | Heavy file-based ETL detection |
| `system.network.io` | `hostmetrics` receiver | Unusual egress detection |
| `system.processes.count` | `hostmetrics` receiver | Hung/zombie process detection |

### Logs

| Log | Source | Purpose |
|---|---|---|
| Application stdout/stderr | `filelog` receiver (`/var/log/jobs/*.log`) | Job output, errors, warnings |
| Python exception tracebacks | `filelog` receiver (multiline) | Failure debugging |
| Job start/end/result events | Structured app logs | Audit trail |
| Cron daemon logs | `journald` receiver | Confirm job was triggered |
| OS syslog | `filelog` receiver (`/var/log/syslog`) | General host health |
| Auth/SSH logs | `filelog` receiver (`/var/log/auth.log`) | Security; detect unauthorized access |

### Traces

| Trace | Source | Purpose |
|---|---|---|
| Job execution root span | OTel auto-instrumentation | Full job duration and outcome |
| Database query spans | OTel auto-instrumentation | SQL latency inside jobs |
| HTTP/API call spans | OTel auto-instrumentation | External API calls (Shopify, Snowflake, etc.) |
| ETL stage spans | OTel Python SDK (manual) | Pipeline stage breakdown |

---

## Collection Strategy

### Agent: `otelcol-contrib` systemd service

Deployed on every DO Droplet. Lightweight config — no heavy processing (that is handled by the Gateway ASG).

### Application Instrumentation: Zero-code auto-instrumentation (preferred)

Jobs are wrapped with `opentelemetry-instrument` — **no code changes required**. This automatically instruments `requests`, `psycopg2`, `pymysql`, `sqlalchemy`, `boto3`, `snowflake-connector`, and other common libraries found in ETL/scheduled jobs.

```bash
# Install once per Droplet
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap --action=install

# Update crontab or systemd unit to wrap job invocation
opentelemetry-instrument \
  --service_name "job-daily-inventory-sync" \
  --exporter_otlp_endpoint "http://localhost:4317" \
  --exporter_otlp_protocol grpc \
  python /opt/jobs/daily_inventory_sync.py
```

For job-level business metrics (success/failure counts, records processed), add ~10 lines to the job entrypoint using the OTel Python SDK `MeterProvider`.

### Connectivity

```
DO Droplet
  └─ WireGuard (wg0) encrypted tunnel
        └─ AWS VPC
              └─ nlb.company.com:443
                    └─ OTel Gateway ASG
```

WireGuard is recommended over IPSec for DO → AWS due to its simplicity, low overhead on Droplets, and native kernel module performance. The OTel agent on each Droplet exports TLS OTLP gRPC to `nlb.company.com:443` through this tunnel.

---

## OTel Collector Agent Config

```yaml
# otelcol-contrib agent for DigitalOcean Droplet (Python scheduled jobs)
# Deploy as: /etc/otelcol-contrib/config.yaml
# Managed by: systemd (otelcol-contrib.service)

extensions:
  health_check:
    endpoint: "0.0.0.0:13133"
  file_storage:
    directory: /var/lib/otelcol/queue
    timeout: 10s

receivers:
  # ── Traces & Metrics from Python OTel SDK ──────────────────────────────
  otlp:
    protocols:
      grpc:
        endpoint: "127.0.0.1:4317"   # localhost only
      http:
        endpoint: "127.0.0.1:4318"

  # ── Host Metrics ────────────────────────────────────────────────────────
  hostmetrics:
    collection_interval: 60s
    scrapers:
      cpu:
        metrics:
          system.cpu.utilization:
            enabled: true
      memory:
        metrics:
          system.memory.utilization:
            enabled: true
      disk: {}
      filesystem:
        exclude_mount_points:
          mount_points: ["/dev", "/sys", "/proc", "/run/lock"]
          match_type: strict
      network: {}
      load: {}
      process:
        mute_process_name_error: true
        mute_process_exe_error: true
        mute_process_io_error: true

  # ── Application Logs ─────────────────────────────────────────────────────
  filelog/app:
    include:
      - /var/log/jobs/*.log
      - /var/log/jobs/**/*.log
    start_at: end
    multiline:
      line_start_pattern: '^\d{4}-\d{2}-\d{2}'   # ISO date prefix
    operators:
      - type: json_parser
        if: 'body matches "^\\{"'
        timestamp:
          parse_from: attributes.timestamp
          layout: '%Y-%m-%dT%H:%M:%S%z'
        severity:
          parse_from: attributes.level
      - type: recombine
        is_last_entry: 'attributes.message not matches "^\\s"'
        combine_field: body
        max_batch_size: 100
    resource:
      service.name: "scheduled-jobs"
      host.location: "digitalocean"

  # ── Cron / systemd Journal ───────────────────────────────────────────────
  journald:
    units:
      - cron
      - crond
    priority: info
    operators:
      - type: add
        field: resource["service.name"]
        value: "cron-daemon"

  # ── Syslog & Auth Logs ───────────────────────────────────────────────────
  filelog/syslog:
    include:
      - /var/log/syslog
      - /var/log/auth.log
    start_at: end
    operators:
      - type: syslog_parser
        protocol: rfc3164
    resource:
      service.name: "os-system"
      host.location: "digitalocean"

processors:
  memory_limiter:
    check_interval: 5s
    limit_mib: 256
    spike_limit_mib: 64

  # Detect hostname via system; DO has no metadata API detector
  resourcedetection:
    detectors: [system, env]
    timeout: 5s
    override: false

  # Brighton-specific resource attributes
  resource:
    attributes:
      - key: deployment.environment
        value: "production"
        action: upsert
      - key: cloud.provider
        value: "digitalocean"
        action: upsert
      - key: team
        value: "data-engineering"
        action: upsert

  batch:
    send_batch_size: 1000
    timeout: 10s

exporters:
  otlp:
    endpoint: "nlb.company.com:443"
    tls:
      insecure: false
      ca_file: "/etc/otelcol/certs/ca.crt"
    headers:
      authorization: "${env:OTEL_GATEWAY_TOKEN}"
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
    sending_queue:
      enabled: true
      num_consumers: 4
      queue_size: 2000
      storage: file_storage   # persists queue across agent restarts

service:
  extensions: [health_check, file_storage]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, resource, batch]
      exporters: [otlp]
    metrics:
      receivers: [otlp, hostmetrics]
      processors: [memory_limiter, resourcedetection, resource, batch]
      exporters: [otlp]
    logs:
      receivers: [otlp, filelog/app, journald, filelog/syslog]
      processors: [memory_limiter, resourcedetection, resource, batch]
      exporters: [otlp]
```

---

## Architecture Flow

```
┌─────────────────────────────────────────────┐
│           DigitalOcean Droplet               │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │   Python Scheduled Job                  │ │
│  │   (cron / systemd timer)                │ │
│  │   + opentelemetry-instrument wrapper    │ │
│  │     → OTLP gRPC → localhost:4317        │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │   otelcol-contrib (systemd service)     │ │
│  │                                          │ │
│  │   Receivers:                            │ │
│  │     otlp · hostmetrics                  │ │
│  │     filelog/app · filelog/syslog        │ │
│  │     journald                            │ │
│  │                                          │ │
│  │   Processors: (lightweight only)        │ │
│  │     memory_limiter · resourcedetection  │ │
│  │     resource · batch                    │ │
│  │                                          │ │
│  │   Exporter:                             │ │
│  │     otlp → nlb.company.com:443 (TLS)   │ │
│  └─────────────────────────────────────────┘ │
└────────────────────┬────────────────────────┘
                     │
              WireGuard VPN
                     │
         ┌───────────▼──────────┐
         │  nlb.company.com:443 │
         │  (AWS NLB)           │
         └───────────┬──────────┘
                     │
         ┌───────────▼──────────┐
         │  OTel Gateway ASG    │
         │  (tail-sample, PII   │
         │   scrub, route)      │
         └──┬──────┬────────┬───┘
            │      │        │
       Prometheus  Loki   Tempo
            └──────┴────────┘
                   │
          AWS Managed Grafana
          grafana.company.com
```

---

## Notes & Assumptions

- **DO Droplet OS**: Ubuntu 22.04 LTS assumed. `journald` and `filelog` receivers cover all standard log paths.
- **No DO metadata API**: DigitalOcean does not expose an instance metadata endpoint compatible with OTel's `resourcedetection` cloud detectors. Use `system` + `env` detectors; set `cloud.provider=digitalocean` manually via the `resource` processor.
- **Queue persistence**: `file_storage` extension ensures the agent's sending queue survives restarts, preventing data loss if the WireGuard tunnel is temporarily unavailable.
- **Heavy processing is not done here**: All tail-sampling, PII scrubbing, and attribute normalization are handled by the Gateway ASG — keeping agent CPU/memory overhead minimal on Droplets.
- **Log path**: `/var/log/jobs/` is assumed. Adjust `filelog/app` include paths to match actual job log output locations.
