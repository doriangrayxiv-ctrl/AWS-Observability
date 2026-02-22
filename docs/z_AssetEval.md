# OTel Collection Strategy: Brighton Collectibles — Systems Assessment

## Methodology

Each system is evaluated against four collection patterns:

| Pattern | When to Use |
|---|---|
| **Remote EC2 Collector Host** | System exposes a scrape/poll endpoint; no agent install possible (SaaS, managed, network device) |
| **Local OTel Agent** | Host is accessible, infrastructure-level telemetry needed, minimal code change desired |
| **Sidecar (container-resident)** | Workload runs in a container orchestrator (EKS, ECS); agent can't be installed on the host OS |
| **Embedded SDK (in-process)** | Application-level traces/metrics/spans; language SDK is the only practical path |

---

## System-by-System Analysis

### 1. Ecommerce — Shopify (SaaS)

| Dimension | Assessment |
|---|---|
| Hosting | Shopify SaaS — no host access |
| Available telemetry | Webhook events, REST/GraphQL API (orders, inventory, etc.) |
| Agent installable? | ❌ No |

**Verdict: Remote EC2 Collector Host**

Shopify does not expose a Prometheus scrape endpoint or accept an agent. The OTel Collector on a dedicated EC2 host must poll Shopify's Admin API using the `httpreceiver` (or a custom scraper pipeline) and receive Shopify webhooks via `otlpreceiver` / `webhookeventreceiver`.

```yaml
# collectors/ec2-remote/shopify-receiver.yaml
receivers:
  httpreceiver/shopify:
    collection_interval: 60s
    endpoint: "https://your-store.myshopify.com/admin/api/2024-01/orders.json"
    headers:
      X-Shopify-Access-Token: "${SHOPIFY_API_TOKEN}"
  webhookeventreceiver:
    endpoint: 0.0.0.0:8088
    path: /shopify/webhook
```

> ⚠️ **Validation Note:** `httpreceiver` is a contrib receiver that parses HTTP responses as metrics. Shopify's API returns JSON, not Prometheus exposition format. A **custom scraper** or **OTel Collector transformation pipeline** using `transformprocessor` + `jsonparseroperator` is required to extract numeric values. This is a known integration gap — flagged for Phase 2 custom pipeline work.

---

### 2. Retail POS — REST API (External)

| Dimension | Assessment |
|---|---|
| Hosting | External system (not AWS-hosted) |
| Available telemetry | REST API polling |
| Agent installable? | ❌ No (external vendor system) |

**Verdict: Remote EC2 Collector Host**

Same pattern as Shopify. The EC2-resident Collector polls the POS REST API on a schedule.

```yaml
# collectors/ec2-remote/pos-receiver.yaml
receivers:
  httpreceiver/pos_api:
    collection_interval: 30s
    endpoint: "https://pos.internal.brighton/api/v1/metrics"
    headers:
      Authorization: "Bearer ${POS_API_TOKEN}"
```

> ✅ **Validation:** `httpreceiver` is valid in `otelcol-contrib`. Polling interval is appropriate for a POS system. TLS assumed on the endpoint.

---

### 3. Intranet Application (JS/PHP on AWS)

| Dimension | Assessment |
|---|---|
| Hosting | AWS EC2 or ECS |
| Language | PHP (server-side), JS (browser) |
| Agent installable? | ✅ Yes (EC2) |
| Code changeable? | Yes — but minimize |

**Verdict: Local OTel Agent (host-level) + Embedded SDK (app-level traces)**

- **Infrastructure metrics** (CPU, memory, disk, network): Local OTel Collector agent using `hostmetricsreceiver`.
- **PHP app traces**: OTel PHP SDK (`open-telemetry/opentelemetry-auto-*` packages) — auto-instrumentation via environment variables, no deep code changes.
- **JS browser traces**: OTel JS Web SDK + browser-side instrumentation sending to a Collector OTLP endpoint.
- **PHP logs**: `filelogreceiver` on the Collector agent reading PHP error/access logs.

```yaml
# collectors/linux-agent/intranet-agent.yaml
receivers:
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu: {}
      memory: {}
      disk: {}
      filesystem: {}
      network: {}
      load: {}
  filelog/php_app:
    include: [/var/log/nginx/access.log, /var/log/php-fpm/error.log]
    operators:
      - type: regex_parser
        regex: '^(?P<remote_addr>\S+) .* \[(?P<time>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+).*" (?P<status>\d{3})'
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
```

> ✅ **Validation:** `hostmetricsreceiver` and `filelogreceiver` are both valid `otelcol-contrib` receivers. PHP auto-instrumentation is available via [opentelemetry-php-contrib](https://github.com/open-telemetry/opentelemetry-php-contrib). This is the correct minimal-change path.

---

### 4. B2B Website (JS/PHP on AWS)

Identical hosting and stack to the Intranet app.

**Verdict: Local OTel Agent + Embedded SDK** — same pattern as Intranet.

> ✅ Reuse the same Collector agent config. Differentiate telemetry using `resourceprocessor` to set `service.name = b2b-website`.

---

### 5. Data/ETL Pipelines (Python on AWS)

| Dimension | Assessment |
|---|---|
| Hosting | AWS EC2 / ECS |
| Language | Python |
| Nature | Long-running or batch pipeline processes |

**Verdict: Embedded SDK (primary) + Local OTel Agent (infrastructure)**

- ETL pipelines are **Python processes** — the OTel Python SDK provides direct span/metric emission tied to pipeline steps (job duration, rows processed, errors). This is the highest-fidelity integration.
- Auto-instrumentation via `opentelemetry-instrument` wrapper requires **zero code changes** for standard libraries (SQLAlchemy, requests, psycopg2, etc.).
- Host infrastructure still monitored by a local Collector agent.

```bash
# Run pipeline with OTel auto-instrumentation — no code changes required
OTEL_SERVICE_NAME=etl-pipeline \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
OTEL_TRACES_EXPORTER=otlp \
OTEL_METRICS_EXPORTER=otlp \
opentelemetry-instrument python pipeline.py
```

> ✅ **Validation:** `opentelemetry-instrument` is the official OTel Python auto-instrumentation entry point. Valid for AWS EC2-hosted Python processes. ECS deployment uses sidecar pattern (see §8).

---

### 6. Scheduled Jobs (Python — AWS + DigitalOcean)

| Dimension | Assessment |
|---|---|
| Hosting | AWS EC2 + DigitalOcean Droplets |
| Language | Python |
| Nature | Short-lived, cron-triggered |

**Verdict: Embedded SDK + Local OTel Agent on each host**

- Short-lived jobs **must** use the embedded SDK because there may not be a persistent Collector agent process available to receive buffered telemetry before job exit. Use `BatchSpanProcessor` with a short `max_export_timeout`.
- Each host (AWS and DO) runs a **local OTel Collector agent** that acts as a local proxy — SDK sends to `localhost:4317`, Collector forwards over VPN/WireGuard to the central stack.

```python
# Minimal SDK setup for short-lived scheduled job
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(endpoint="http://localhost:4317"),
        max_export_batch_size=512,
        export_timeout_millis=5000,   # critical for short-lived jobs
    )
)
trace.set_tracer_provider(provider)

# REQUIRED: explicit shutdown to guarantee flush before process exit
provider.shutdown()
```

> ⚠️ **Validation Note:** For very short jobs (< 5s), `BatchSpanProcessor` may not flush before process exit. Always call `provider.shutdown()` explicitly at job end, or switch to `SimpleSpanProcessor` for guaranteed synchronous export. **Mark as implementation risk** — test plan required.

---

### 7. AWS EKS (Kubernetes)

| Dimension | Assessment |
|---|---|
| Hosting | AWS EKS |
| Workloads | Mixed (likely include Intranet, B2B, ETL containers) |
| Agent installable on node? | ✅ DaemonSet |
| Per-pod telemetry? | ✅ Sidecar |

**Verdict: Sidecar (per-pod) + DaemonSet (node-level)**

- **DaemonSet Collector**: Node-level metrics (`hostmetricsreceiver`), Kubernetes API metrics (`k8sclusterreceiver`), node logs.
- **Sidecar Collector**: Application OTLP traffic from SDK-instrumented containers. Each pod gets a Collector sidecar receiving on `localhost:4317`.

```yaml
# k8s/otelcol-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otelcol-agent
  namespace: observability
spec:
  selector:
    matchLabels:
      app: otelcol-agent
  template:
    metadata:
      labels:
        app: otelcol-agent
    spec:
      serviceAccountName: otelcol-agent
      containers:
        - name: otelcol
          image: otel/opentelemetry-collector-contrib:0.96.0
          args: ["--config=/conf/otelcol-config.yaml"]
          volumeMounts:
            - name: config
              mountPath: /conf
            - name: varlog
              mountPath: /var/log
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: otelcol-agent-config
        - name: varlog
          hostPath:
            path: /var/log
```

> ✅ **Validation:** `k8sclusterreceiver` and `kubeletstatsreceiver` are valid contrib receivers. DaemonSet requires `ClusterRole` with `get/list/watch` on nodes, pods, and events. RBAC manifest required — flagged for Phase 1 IaC deliverable.

---

### 8. AWS ECS Fargate

| Dimension | Assessment |
|---|---|
| Hosting | AWS ECS Fargate |
| Agent on host OS? | ❌ Fargate — no host access |
| Sidecar possible? | ✅ Yes — ECS task definition sidecar container |

**Verdict: Sidecar**

Fargate abstracts the host entirely. The only viable Collector pattern is a sidecar container in the same ECS Task Definition. The `awsecscontainermetricsreceiver` provides Fargate task-level CPU/memory from the ECS task metadata endpoint.

```json
{
  "family": "app-with-otel",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "brighton/app:latest",
      "environment": [
        {"name": "OTEL_EXPORTER_OTLP_ENDPOINT", "value": "http://localhost:4317"},
        {"name": "OTEL_SERVICE_NAME",            "value": "brighton-app"}
      ]
    },
    {
      "name": "otelcol-sidecar",
      "image": "otel/opentelemetry-collector-contrib:0.96.0",
      "essential": false,
      "command": ["--config=/etc/otelcol/config.yaml"],
      "portMappings": [
        {"containerPort": 4317, "protocol": "tcp"}
      ]
    }
  ]
}
```

> ✅ **Validation:** ECS sidecar pattern is the canonical approach for Fargate. The `awsecscontainermetricsreceiver` is a valid contrib receiver for task-level telemetry.

---

### 9. AWS Lambda

| Dimension | Assessment |
|---|---|
| Hosting | AWS Lambda (serverless) |
| Agent installable? | ❌ No persistent process |
| Sidecar possible? | ❌ No container model |

**Verdict: Embedded SDK via Lambda Layer**

AWS Lambda requires the OTel Lambda Layer — a packaged Lambda Extension that runs the Collector as a Lambda extension process alongside the function handler.

```yaml
# SAM/CloudFormation — Lambda with OTel Layer
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Layers:
        - !Sub "arn:aws:lambda:${AWS::Region}:184161586896:layer:opentelemetry-collector-arm64-0_9_0:1"
      Environment:
        Variables:
          OTEL_SERVICE_NAME: brighton-lambda-fn
          OPENTELEMETRY_COLLECTOR_CONFIG_FILE: /var/task/collector.yaml
          OTEL_EXPORTER_OTLP_ENDPOINT: https://otelcollector.brighton.internal:4317
```

> ✅ **Validation:** `arn:aws:lambda:...:184161586896:layer:opentelemetry-collector-*` is the official AWS-managed OTel Lambda layer ARN. The layer runs `otelcol-contrib` as a Lambda Extension. Cold start latency impact should be measured — flagged for performance testing.

---

### 10. AWS RDS (Postgres / MySQL)

| Dimension | Assessment |
|---|---|
| Hosting | AWS RDS (managed) |
| Agent on DB host? | ❌ No — managed service |
| Metrics available? | ✅ CloudWatch + DB-level via OTel receivers |

**Verdict: Remote EC2 Collector Host (dual approach)**

- **CloudWatch metrics** (instance-level): `awscloudwatchreceiver` on EC2 Collector host.
- **DB-level metrics** (queries, replication lag, connections): `postgresqlreceiver` / `mysqlreceiver` connecting to the RDS endpoint from the EC2 Collector. Requires network path and a read-only DB monitoring user.

```yaml
# collectors/ec2-remote/rds-receivers.yaml
receivers:
  awscloudwatch:
    region: us-east-1
    poll_interval: 1m
    metrics:
      names:
        - AWS/RDS/CPUUtilization
        - AWS/RDS/DatabaseConnections
        - AWS/RDS/ReadLatency
        - AWS/RDS/WriteLatency
        - AWS/RDS/FreeStorageSpace

  postgresql:
    endpoint: "${RDS_POSTGRES_HOST}:5432"
    username: "${RDS_MONITOR_USER}"
    password: "${RDS_MONITOR_PASSWORD}"
    databases: ["brighton_ecommerce", "brighton_b2b"]
    tls:
      insecure: false
      ca_file: /etc/ssl/certs/rds-ca.pem

  mysql:
    endpoint: "${RDS_MYSQL_HOST}:3306"
    username: "${RDS_MONITOR_USER}"
    password: "${RDS_MONITOR_PASSWORD}"
    collection_interval: 60s
```

> ✅ **Validation:** `awscloudwatchreceiver`, `postgresqlreceiver`, and `mysqlreceiver` are all valid `otelcol-contrib` receivers. The EC2 Collector host must reside in the same VPC (or VPC-peered) with RDS subnets. Security group rule required: RDS SG must allow inbound 5432/3306 from the Collector EC2 instance.

---

### 11. On-Prem Databases (Postgres / MySQL)

**Verdict: Local OTel Agent**

Deploy `otelcol-contrib` on the database server (or a nearby monitoring VM on the same LAN). Use `postgresqlreceiver` / `mysqlreceiver` connecting to `localhost`. Telemetry forwarded to the central stack over VPN.

> ✅ Same receiver config as §10 above — change endpoint to `localhost` or LAN IP.

---

### 12. Snowflake Data Warehouse (SaaS)

| Dimension | Assessment |
|---|---|
| Hosting | Snowflake SaaS |
| Agent installable? | ❌ No |

**Verdict: Remote EC2 Collector Host — via Prometheus exporter workaround**

> ❌ **Validation Issue — MARKED PROBLEM:**
>
> There is **no stable `snowflakereceiver` in `otelcol-contrib`** as of February 2026. The receiver was listed as experimental and has not been promoted to a stable contrib component.
>
> **Viable workarounds (in priority order):**
>
> 1. Deploy the [Grafana snowflake-prometheus-exporter](https://github.com/grafana/snowflake-prometheus-exporter) on the EC2 Collector host. Scrape it with `prometheusreceiver` in the Collector, then remote-write to Prometheus.
> 2. Run a scheduled Python script (OTel Python SDK) that queries Snowflake's `ACCOUNT_USAGE` schema and emits metrics via OTLP push.
> 3. Use the Snowflake REST API via `httpreceiver` with a custom `transformprocessor` pipeline to parse query/usage JSON.
>
> **Recommendation:** Option 1 (Grafana exporter + `prometheusreceiver`) for Phase 2. Flag for dedicated spike/implementation task.

---

## Consolidated Assignment Matrix

| System | EC2 Remote Collector | Local OTel Agent | Sidecar | Embedded SDK |
|---|:---:|:---:|:---:|:---:|
| Shopify (SaaS) | ✅ Primary | | | |
| Retail POS (External API) | ✅ Primary | | | |
| Intranet App (AWS EC2/ECS) | | ✅ Infra | ✅ If ECS | ✅ App traces |
| B2B Website (AWS EC2/ECS) | | ✅ Infra | ✅ If ECS | ✅ App traces |
| ETL Pipelines (Python/AWS) | | ✅ Infra | ✅ If ECS | ✅ Primary |
| Scheduled Jobs (Python) | | ✅ Agent proxy | | ✅ Primary |
| AWS EKS | | ✅ DaemonSet | ✅ Per-pod | ✅ App traces |
| AWS ECS Fargate | | | ✅ Primary | ✅ App traces |
| AWS Lambda | | | | ✅ Lambda Layer |
| AWS RDS | ✅ Primary | | | |
| On-Prem DB (PG/MySQL) | | ✅ Primary | | |
| Snowflake (SaaS) | ✅ Via Prom exporter ⚠️ | | | |

---

## Validation Summary

| # | System | Status | Notes |
|---|---|---|---|
| 1 | Shopify | ⚠️ | `httpreceiver` requires custom JSON→metric transform pipeline |
| 2 | POS API | ✅ | Standard `httpreceiver` polling; TLS on endpoint assumed |
| 3 | Intranet App | ✅ | Agent + PHP auto-instrumentation via `opentelemetry-php-contrib` |
| 4 | B2B Website | ✅ | Same pattern as Intranet; differentiate via `resourceprocessor` |
| 5 | ETL Pipelines | ✅ | `opentelemetry-instrument` auto-wrap; zero code changes |
| 6 | Scheduled Jobs | ⚠️ | `BatchSpanProcessor` flush risk on short-lived jobs — must call `provider.shutdown()` |
| 7 | EKS | ✅ | DaemonSet + RBAC ClusterRole required; flagged for IaC Phase 1 |
| 8 | ECS Fargate | ✅ | Sidecar with `awsecscontainermetricsreceiver` |
| 9 | Lambda | ✅ | Official AWS OTel Layer; cold-start impact requires performance test |
| 10 | RDS | ✅ | EC2 Collector + VPC network path + SG rules required |
| 11 | On-Prem DB | ✅ | Local agent; telemetry forwarded over VPN |
| 12 | Snowflake | ❌ | No stable `snowflakereceiver` in contrib — Grafana Prom exporter workaround required |

---

*Last updated: 2026-02-20*