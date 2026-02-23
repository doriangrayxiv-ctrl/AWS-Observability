# OTel Collection — Endpoint Architecture

Defines how `otelcol-contrib` is deployed and configured across each collection scenario. All agents forward telemetry via TLS OTLP to the Gateway NLB at `nlb.company.com:443`. Agents perform **lightweight processing only** — PII scrubbing, deduplication, filtering, and batching — before data leaves the host. **Tail sampling is performed exclusively at the Gateway tier**, where the complete trace can be assembled from all contributing agents before a sampling decision is made.

---

## Common Components

Shared pipeline configuration applied across all `otelcol-contrib` agent and sidecar deployments. The processor ordering, enrichment rules, and TLS exporter settings defined here are applied by every agent and sidecar section below before telemetry is forwarded to the Gateway NLB at `nlb.company.com:443`.


---

### Processing

All agent deployments run the following processor stages in the collector pipeline before export. Processor configs are rendered from templates and deployed alongside receiver/exporter configs through the same DevOps pipeline.

#### Tail Sampling (Gateway tier only)

> **Tail sampling does not run on agents.** A single agent only sees the spans it locally collected and cannot make trace-complete sampling decisions. Tail sampling is applied at the **OTel Gateway Collectors**, where the full trace is assembled from all contributing agents before a sampling decision is made.

- Implemented via `tailsamplingprocessor` on **Gateway Collector instances only**.
- Sampling decisions are made after a configurable wait window (e.g., 10–30s) once the full trace is assembled at the Gateway.
- Policies are defined per service: always-sample on errors and slow spans; probabilistic rate on healthy traces.
- Reduces trace volume forwarded to Tempo without losing signal on anomalous requests.
- Agent configurations do **not** include `tailsamplingprocessor`; agents forward all spans unsampled to the Gateway.

#### PII Scrubbing

- Implemented via `redactionprocessor` (otelcol-contrib).
- Configured with block-listed attribute keys (e.g., `user.email`, `http.request.header.authorization`, `db.statement` patterns matching card/SSN regexes).
- Matching attribute values are replaced with a redacted placeholder before the span/log leaves the host.
- Scrubbing rules are maintained centrally in the pipeline config template and versioned in the infrastructure repo.

#### Deduplication

- Implemented via `filterprocessor` with identity-based drop conditions.
- Duplicate log lines emitted by applications on retry or multi-writer scenarios are suppressed using attribute match rules (e.g., matching on `log.record.uid` or a stable hash attribute set by the app).
- Metric deduplication is handled upstream by Prometheus remote-write dedup on the backend; the agent does not attempt metric-level dedup.

#### Filtering

- Implemented via `filterprocessor`.
- Drop rules applied per signal type:
  - **Logs**: suppress DEBUG-level logs in production; drop known-noisy log sources by `service.name` or log body pattern.
  - **Metrics**: drop high-cardinality or unused metric names defined in a blocklist.
  - **Traces**: drop health-check and synthetic monitor spans (matched by `http.target` or `http.url` patterns).
- Filter rules are environment-aware (e.g., DEBUG logs retained in staging, dropped in prod) via pipeline-injected variables.

### Common Processor Stack

All `otelcol-contrib` agent and sidecar deployments share the following six-processor pipeline. Per-deployment sections below list only **additions to or exclusions from** this stack.

| Component | Purpose |
|-----------|---------|
| `memorylimiterprocessor` | Prevent OOM if ingest spikes beyond local buffer capacity |
| `batchprocessor` | Batch spans/metrics/logs before export to reduce NLB connections |
| `resourcedetectionprocessor` | Auto-attach cloud/host metadata (instance ID, region, AZ, hostname — or ECS task ARN / K8s node name depending on environment) |
| `attributesprocessor` | Attach static labels: `env`, `service.name`, `team` (from config template vars or task definition env vars) |
| `redactionprocessor` | PII scrubbing — block-listed attribute keys replaced with a redacted placeholder before export |
| `filterprocessor` | Drop debug logs (prod), unused metrics, health-check/readiness-probe spans, and duplicate records per environment rules |

---

## Agent Deployments

Persistent `otelcol-contrib` processes deployed on each host, node, or dedicated EC2 instance. All agent deployments apply the [Common Processor Stack](#common-processor-stack) and forward telemetry via TLS OTLP to the Gateway NLB. No tail sampling occurs at the agent — all spans are forwarded unsampled for Gateway-tier decisions.


---

### EC2 — Standard Host Agent (Linux & Windows)

#### Covered Systems

| System | OS | Signal Types |
|--------|----|-------------|
| Intranet (JS/PHP) | Linux | Logs · Traces · Metrics |
| B2B Website (JS/PHP) | Linux | Logs · Traces · Metrics |
| PostgreSQL | Linux | Logs · Metrics |
| MySQL | Linux / Windows | Logs · Metrics |
| Windows Application Hosts | Windows | Logs · Metrics |

#### Deployment Model

One `otelcol-contrib` process per host, running as a **systemd service** (Linux) or **Windows Service** (Windows). The binary is a custom `otelcol-contrib` build compiled with only the required components (see below) to minimize binary size and attack surface.

Configurations are **templated and deployed via DevOps pipelines** (e.g., Ansible, AWS SSM, or a CI/CD pipeline). A host-specific variable file supplies environment, role, and service identifiers that are injected into the collector config at deploy time.

#### Custom otelcol-contrib Build — Included Components

> Built using the [OpenTelemetry Collector Builder (`ocb`)](https://github.com/open-telemetry/opentelemetry-collector/tree/main/cmd/builder). Only components actually used are compiled in.

##### Receivers

| Component | Purpose |
|-----------|---------|
| `hostmetricsreceiver` | CPU, memory, disk, network, filesystem per host |
| `filelogreceiver` | Application log files (structured + unstructured); tail mode |
| `otlpreceiver` | Accept traces and metrics pushed by language SDKs (JS/PHP/Python apps) |
| `postgresqlreceiver` | PostgreSQL database metrics (connections, commits, table/index stats) |
| `mysqlreceiver` | MySQL database metrics (threads, queries, InnoDB stats) |
| `windowsperfcountersreceiver` | *(Windows only)* OS-level performance counters |
| `windowseventlogreceiver` | *(Windows only)* Windows Event Log (Application, System, Security channels) |

##### Processors

All six components from the [Common Processor Stack](#common-processor-stack) — no additions or exclusions for this deployment.

##### Exporters

| Component | Purpose |
|-----------|------|
| `otlpexporter` | Forward all signals (gRPC, TLS) to `nlb.company.com:443` |

##### Extensions

| Component | Purpose |
|-----------|---------|
| `healthcheckextension` | Local `/health` endpoint for pipeline monitoring |
| `zpagesextension` | Debug traces/pipelines during rollout |

#### Signal Collection Detail

**Metrics**
- Host-level OS metrics via `hostmetricsreceiver` (CPU, memory, disk I/O, net I/O).
- Database metrics via `postgresqlreceiver` / `mysqlreceiver` (scrape interval: 60s).
- Application metrics emitted by OTel SDK → received on local `otlpreceiver`.

**Logs**
- Application log files tailed by `filelogreceiver`. Operators applied for log parsing (regex or JSON parsing depending on app format).
- Database slow-query logs and error logs tailed from known file paths.
- Windows Event Log (Windows hosts only) via `windowseventlogreceiver`.

**Traces**
- Applications instrumented with OTel language SDK (JS/PHP: `@opentelemetry/sdk-node` or contrib auto-instrumentation) push spans to local `otlpreceiver` on `localhost:4317`.
- No sampling at the agent; full trace data forwarded to Gateway for tail-sampling decisions.

#### Deployment Pipeline Notes

- Config templates live in the observability infrastructure repo.
- Role-specific variable files (`ec2-linux-db.vars`, `ec2-windows-app.vars`, etc.) drive `SERVICE_NAME`, log paths, and which receivers are enabled.
- Ansible or AWS SSM Run Command renders the template and restarts the collector service.
- Post-deploy health check hits `localhost:13133/health`; pipeline fails if non-200.
- Binary version is pinned in the pipeline; upgrades follow the same template render + restart flow.

---

### EC2 — Remote Collection Scraper (Dedicated Instance)

A single dedicated EC2 instance (`t3.medium` or similar) running `otelcol-contrib` configured exclusively to pull telemetry from **external and managed services** that cannot host their own agent. This instance has no local application workload.

> This isolates external API credentials and polling schedules from general-purpose hosts and allows independent scaling and security controls.

#### Covered Systems

| System | Collection Method | Signal Types |
|--------|------------------|-------------|
| Shopify (Ecommerce SaaS) | Webhook receiver — Shopify pushes order/event data to `httplogreceiver` | Logs · Metrics |
| Retail POS (REST API) | `httplogreceiver` webhook inbound (if POS supports webhooks); Python cron script → `filelogreceiver` for pull-only APIs | Logs · Metrics |
| AWS RDS (MySQL) | `awscloudwatchreceiver` (native RDS metrics + slow-query logs from CW Logs) | Logs · Metrics |
| Snowflake (Data Warehouse) | `snowflakereceiver` (Snowflake REST API / information_schema queries) | Metrics |
| AWS CloudWatch | `awscloudwatchreceiver` (pull metrics + log groups from CW) | Logs · Metrics |

#### Receiver Detail

##### Shopify
- Shopify does not expose a native OTel endpoint. The recommended path is a **Shopify webhook** configured to POST order/event payloads to an `httplogreceiver` listener on this instance.
- The `httplogreceiver` accepts inbound HTTP POST bodies and emits one log record per payload; no custom parsing code required in the collector.
- Fallback (if webhooks cannot be used): a Python script on cron calls the Shopify Admin REST API, appends newline-delimited JSON records to `/var/otel/shopify_events.ndjson`, and `filelogreceiver` tails that file.
- **Credential**: Shopify Webhook HMAC secret stored in AWS Secrets Manager; mounted as env var.

##### Retail POS (REST API)
- If the POS system supports webhooks: configure the POS to push events to an `httplogreceiver` endpoint on this instance.
- If the POS is pull-only: a Python script on cron calls the POS REST API and appends newline-delimited JSON records to `/var/otel/pos_events.ndjson`; `filelogreceiver` tails that file and emits log records into the pipeline.
- Exact collection method depends on POS vendor capability — webhook preferred; cron poller as fallback.
- **Credential**: POS API key stored in AWS Secrets Manager.

##### AWS RDS
- RDS metrics are natively published to Amazon CloudWatch. The `awscloudwatchreceiver` pulls these on a configurable interval.
- RDS slow-query logs and error logs are published to CloudWatch Logs; pulled via the same receiver using log group name patterns.
- IAM Role attached to this EC2 instance grants `cloudwatch:GetMetricStatistics`, `cloudwatch:ListMetrics`, `logs:FilterLogEvents` scoped to RDS log groups.
- No direct DB connectivity required from this instance.

##### Snowflake
- The `snowflakereceiver` (otelcol-contrib) queries Snowflake's `SNOWFLAKE.ACCOUNT_USAGE` schema for warehouse metrics, query history, storage, and credit usage.
- Requires a dedicated Snowflake service account with `MONITOR` privilege on the `SNOWFLAKE` database.
- **Credential**: Snowflake username/password or key-pair auth stored in AWS Secrets Manager.
- Poll interval: 300s (Snowflake latency for `ACCOUNT_USAGE` views is ~45 min; fine-grained real-time metrics are not available via this path).

##### AWS CloudWatch (General)
- Catch-all for any AWS services not covered by a dedicated receiver.
- Configured with namespace filters (e.g., `AWS/EC2`, `AWS/ELB`, `AWS/SQS`, `AWS/Lambda`).
- Also used to route CloudWatch Log group streams (Lambda stdout, VPC Flow Logs, etc.) into Loki.
- IAM Role: `cloudwatch:GetMetricData`, `cloudwatch:ListMetrics`, `logs:DescribeLogGroups`, `logs:FilterLogEvents`.

#### Custom otelcol-contrib Build — Included Components

##### Receivers

| Component | Purpose |
|-----------|---------|
| `httplogreceiver` | Inbound webhook payloads (Shopify, POS) — emits one log record per HTTP POST body |
| `filelogreceiver` | Tails NDJSON output files written by Python cron pollers (POS pull-only, Shopify fallback) |
| `otlpreceiver` | Reserved for any future SDK-instrumented local processes |
| `awscloudwatchreceiver` | Pull metrics + logs from Amazon CloudWatch (RDS, Lambda, general AWS) |
| `snowflakereceiver` | Pull Snowflake account usage metrics |

##### Processors

All six components from the [Common Processor Stack](#common-processor-stack). The `attributesprocessor` additionally attaches a `source.system` label (e.g., `shopify`, `rds`, `snowflake`) via config template vars to identify the originating system on every record.

##### Exporters

| Component | Purpose |
|-----------|------|
| `otlpexporter` | Forward all signals (gRPC, TLS) to `nlb.company.com:443` |

#### Security & Credentials

- IAM Instance Profile on this EC2 grants scoped CloudWatch read permissions — no long-lived AWS keys needed.
- All other secrets (Snowflake, POS API key, Shopify HMAC) are fetched at startup from **AWS Secrets Manager** via the `env` provider or a startup script.
- Security Group restricts inbound `httplogreceiver` webhook ports to known source IP ranges (Shopify IP ranges, POS datacenter IPs). Specific port assignments are defined during detailed config — out of scope for this high-level design.
- No inbound SSH; use AWS SSM Session Manager for access.

---

### EKS DaemonSet

The primary collection method for AWS EKS workloads. `otelcol-contrib` is deployed as a Kubernetes **DaemonSet**, placing one collector pod on every node in the cluster. This mirrors the EC2 host agent pattern at the node level and provides automatic coverage for all pods running on a node without requiring per-pod configuration changes.

An optional **per-pod sidecar** supplements the DaemonSet for workloads requiring pod-level processing isolation. It is not the default and should not be deployed cluster-wide.

---

#### Covered Systems

| System | Hosting | Signal Types |
|--------|---------|-------------|
| Application pods | AWS EKS (all node types) | Logs · Traces · Metrics |
| Kubernetes cluster state | AWS EKS | Metrics |

---

#### DaemonSet vs. Sidecar — Decision Rule

| Pattern | When to Use |
|---------|-------------|
| **DaemonSet only** | Default for all workloads. App pods push OTLP to the node-local DaemonSet collector via the node's host IP. |
| **DaemonSet + Sidecar** | Workloads requiring pod-level processing isolation, or where routing OTLP through a shared node collector is unacceptable for security or compliance reasons. |

Do not deploy sidecars cluster-wide — DaemonSet coverage is sufficient for the vast majority of services and reduces resource and operational overhead significantly.

---

#### Telemetry Flow

```
┌─── EKS Node ──────────────────────────────────────────────────────────┐
│                                                                        │
│  ┌─── Pod A ────────────────┐    ┌─── Pod B (isolated) ─────────────┐ │
│  │  App Container           │    │  App Container                   │ │
│  │  (OTel SDK / auto-instr) │    │  (OTel SDK / auto-instr)         │ │
│  │                          │    │  ┌───────────────────────────┐   │ │
│  │  Traces/Metrics → OTLP   │    │  │  otelcol-contrib Sidecar  │   │ │
│  │  Logs    → stdout        │    │  │  (optional, per-pod)      │   │ │
│  └──────────┬───────────────┘    │  │  receives on localhost:   │   │ │
│             │ OTLP               │  │  4317, forwards to        │   │ │
│             │ hostIP:4317        │  │  DaemonSet or Gateway NLB │   │ │
│             │                    │  └─────────────┬─────────────┘   │ │
│             │                    └────────────────┼─────────────────┘ │
│             │                                     │ TLS OTLP          │
│             └─────────────────────┬───────────────┘                   │
│                                   ▼                                   │
│              ┌────────────────────────────────────────────────────┐   │
│              │            otelcol-contrib DaemonSet Pod           │   │
│              │                                                    │   │
│              │  otlpreceiver        ◄── OTLP from app pods        │   │
│              │  hostmetricsreceiver ◄── Node OS (CPU/mem/disk)    │   │
│              │  kubeletstatsreceiver◄── Per-pod/container metrics │   │
│              │  k8sclusterreceiver  ◄── Cluster state (1 pod)     │   │
│              │  filelogreceiver     ◄── /var/log/pods/**/*.log    │   │
│              │                                                    │   │
│              │  k8sattributesprocessor  (pod/ns/deployment labels)│   │
│              │  resourcedetectionprocessor (EC2 node metadata)    │   │
│              │  attributesprocessor     (env, team, cluster.name) │   │
│              │  redactionprocessor      (PII scrub)               │   │
│              │  filterprocessor         (debug logs, healthchecks)│   │
│              │  memorylimiterprocessor + batchprocessor           │   │
│              └───────────────────────┬────────────────────────────┘   │
└──────────────────────────────────────┼────────────────────────────────┘
                                       │ TLS OTLP gRPC
                                       ▼
                           nlb.company.com:443
                           (Gateway NLB → ASG)
                                       │
               ┌───────────────────────┼────────────────────┐
               ▼                       ▼                     ▼
          Prometheus                  Loki                 Tempo
          (metrics)                  (logs)               (traces)
               └───────────────────────┴─────────────────────┘
                                       ▼
                           grafana.company.com:443
                           (WAF → ALB OIDC → Managed Grafana)
```

---

#### Signal Flow Detail

##### Metrics

| Source | Receiver | What Is Captured |
|--------|----------|-----------------|
| Node OS | `hostmetricsreceiver` | CPU, memory, disk I/O, network I/O — same scrape set as EC2 host agent |
| Kubelet stats | `kubeletstatsreceiver` | Per-pod and per-container CPU, memory, network, filesystem — pulled from the Kubelet `/stats/summary` endpoint on each node |
| Cluster state | `k8sclusterreceiver` | Deployment replica counts, pod phase counts, node conditions, HPA state — **runs on one DaemonSet pod only**, elected via a `k8sleaderelector` extension to prevent duplicate cluster-level metrics |
| App OTel SDK | `otlpreceiver` | Custom business and runtime metrics emitted by instrumented pods; received on `hostIP:4317` |

All metrics → `DaemonSet → Gateway NLB → Prometheus remote-write`.

##### Logs

| Source | Receiver | What Is Captured |
|--------|----------|-----------------|
| Pod stdout/stderr | `filelogreceiver` | Tails `/var/log/pods/*/*/*.log` mounted from the node's filesystem via `hostPath`; covers all containers on the node automatically |
| Node system logs | `filelogreceiver` | `/var/log/messages` or `/var/log/syslog` depending on node OS |

The `filelogreceiver` operators parse the Kubernetes container log wrapper format (JSON with `log`, `time`, `stream` fields) to extract the inner log body. `k8sattributesprocessor` enriches every log record with `k8s.namespace.name`, `k8s.pod.name`, `k8s.container.name`, and `k8s.deployment.name` by correlating the file path against the Kubernetes API.

All logs → `DaemonSet → Gateway NLB → Grafana Loki`.

##### Traces

| Source | Receiver | What Is Captured |
|--------|----------|-----------------|
| App pods (SDK / auto-instr) | `otlpreceiver` | Spans pushed from app containers to the node's host IP on port 4317; pod resolves the address via the Kubernetes Downward API (see below) |
| Optional sidecar | `otlpreceiver` | For isolated pods: sidecar receives on `localhost:4317`, applies pod-level processing, then forwards to the DaemonSet or directly to the Gateway NLB |

App pods reference the node IP at runtime using the **Kubernetes Downward API** so no hardcoded addresses or service discovery is required:

```yaml
env:
  - name: NODE_IP
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://$(NODE_IP):4317"
  - name: OTEL_SERVICE_NAME
    value: "my-service"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment=prod,team=platform"
```

> Tail sampling is **not** applied at the DaemonSet. Full trace data is forwarded to the Gateway, where tail-sampling decisions are made across the complete trace — consistent with all other agent deployments.

All traces → `DaemonSet → Gateway NLB → Grafana Tempo`.

---

#### Custom otelcol-contrib Build — Included Components

##### Receivers

| Component | Purpose |
|-----------|---------|
| `otlpreceiver` | Receive traces and metrics from app pods via `hostIP:4317/4318` |
| `hostmetricsreceiver` | Node OS metrics (CPU, mem, disk, net) |
| `kubeletstatsreceiver` | Per-pod and per-container resource metrics from the Kubelet stats endpoint |
| `k8sclusterreceiver` | Cluster-level metrics (deployments, pod phases, node conditions, HPA) — leader pod only |
| `filelogreceiver` | Pod stdout/stderr logs from `/var/log/pods`; node system logs |

##### Processors

All six components from the [Common Processor Stack](#common-processor-stack), plus one EKS-specific addition:

| Addition | Purpose |
|----------|--------|
| `k8sattributesprocessor` | Enrich spans and log records with pod, namespace, and deployment metadata via K8s API |

The `resourcedetectionprocessor` is configured for EC2 node metadata (instance ID, region, AZ) and Kubernetes node name.

##### Exporters

| Component | Purpose |
|-----------|------|
| `otlpexporter` | Forward all signals (gRPC, TLS) to `nlb.company.com:443` |

##### Extensions

| Component | Purpose |
|-----------|---------|
| `healthcheckextension` | `/health` endpoint used as the DaemonSet pod liveness probe |
| `zpagesextension` | Debug pipeline visibility during rollout |
| `k8sleaderelector` | Elect one DaemonSet pod to run `k8sclusterreceiver`; all other pods skip it |

---

#### RBAC Requirements

The DaemonSet ServiceAccount requires a ClusterRole so `k8sattributesprocessor` and `k8sclusterreceiver` can query the Kubernetes API server:

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces", "nodes", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "daemonsets", "statefulsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch"]
```

---

#### Deployment Notes

- **`hostPath` mounts**: `/var/log/pods` and `/var/log` must be mounted read-only into the DaemonSet pod for `filelogreceiver` access.
- **`hostPort`**: The `otlpreceiver` listens on `hostPort: 4317` and `hostPort: 4318` so app pods can reach it via the node IP injected by the Downward API. Using `hostNetwork: true` is an alternative but exposes broader node networking — prefer `hostPort`.
- **Resource limits**: Set CPU `request: 100m / limit: 500m`, memory `request: 200Mi / limit: 400Mi`. Mark the pod with `priorityClassName: system-node-critical` to reduce eviction risk under node memory pressure.
- **Config management**: Config stored in a `ConfigMap` and mounted into the DaemonSet pod. Updates are applied via `kubectl rollout restart daemonset/otelcol-contrib` from the same DevOps pipeline used for EC2 agent deployments.
- **TLS CA cert**: Mount the Gateway NLB CA certificate into the pod from a Kubernetes `Secret` and reference it in the `otlpexporter` TLS stanza.
- **Tolerations**: Add tolerations for control-plane node taints if node-level metrics from control plane nodes are required.
- **Node selector**: Scope to Linux nodes only (`kubernetes.io/os: linux`) if Windows node pools are present in the cluster.

---

## Sidecar Deployments

The primary collection method for containerized workloads running on **AWS ECS (Fargate and EC2 launch type)**. An `otelcol-contrib` container is co-deployed alongside each application container within the same ECS task, sharing the task's network namespace. This is the only viable agent pattern for Fargate, where no host OS access exists.

---

### How Many Sidecars?

**One sidecar container per ECS task definition** — not one per cluster or per service.

Because Fargate tasks are isolated at the network level, a single centralized collector cannot reach other tasks' localhost interfaces. Each task must carry its own collector sidecar to capture that task's OTLP output and container metrics. This means sidecar count scales linearly with running task count, which is expected and manageable: the sidecar is lightweight (128–256 MB RAM, minimal CPU at idle) and its resource reservation is defined in the task definition alongside the app container.

For **ECS on EC2** (non-Fargate), a host-level agent (see [EC2 — Standard Host Agent](#ec2--standard-host-agent-linux--windows)) is preferred over per-task sidecars for resource efficiency. Sidecars are still valid in EC2 launch type if task-level isolation is required.

---

### Covered Systems

| System | Hosting | Launch Type | Signal Types |
|--------|---------|-------------|-------------|
| Application containers | AWS ECS | Fargate | Logs · Traces · Metrics |
| Application containers | AWS ECS | EC2 (sidecar variant) | Logs · Traces · Metrics |

---

### Sidecar Architecture

Each ECS task definition is updated to include `otelcol-contrib` as a second container entry. The two containers share a network namespace, so the app container emits OTLP to `localhost:4317` / `localhost:4318` and the sidecar receives it without any cross-task or cross-host networking.

```
┌─── ECS Task ───────────────────────────────────────────┐
│                                                         │
│  ┌─────────────────────┐   OTLP (localhost:4317)       │
│  │   App Container     │ ─────────────────────────►    │
│  │  (your service)     │   traces · metrics · logs     │
│  │                     │   (via OTel SDK imports)      │
│  └─────────────────────┘                               │
│                                                         │
│  ┌─────────────────────┐                               │
│  │  otelcol-contrib    │ ──► TLS OTLP ──► Gateway NLB  │
│  │  Sidecar            │         nlb.company.com:443   │
│  └─────────────────────┘                               │
└─────────────────────────────────────────────────────────┘
```

The sidecar:
- Receives **traces, metrics, and logs** from the app container via `otlpreceiver` on `localhost:4317/4318`. All three signals share the same OTLP transport — the app uses `from opentelemetry import trace`, `metrics`, and `logs` and points its exporter at `localhost:4317`.
- Collects ECS task-level CPU, memory, and network metrics via `awsecscontainermetricsreceiver`.
- Applies the standard processor stack (resource detection, attribute enrichment, PII scrubbing, batching).
- Exports all signals via TLS OTLP to the Gateway NLB.

> **Note on container stdout:** App container stdout goes to CloudWatch Logs via the `awslogs` log driver — this is independent of the sidecar and is not collected by it. The sidecar receives logs exclusively via OTLP from the OTel SDK. Do not configure `filelogreceiver` pointing at `/dev/stdout` or any path derived from another container's stdout; that path is not accessible across containers in a Fargate task.

---

### Sidecar Container Requirements

| Setting | Value |
|---------|-------|
| Image | `otel/opentelemetry-collector-contrib:<pinned-version>` |
| CPU reservation | 64–128 CPU units |
| Memory reservation | 128–256 MiB |
| Essential | `false` — app container should not stop if collector crashes |
| Mount | Shared config volume or baked-in config via SSM Parameter / S3 at startup |
| Log driver | `awslogs` (sidecar's own logs → CloudWatch for operator visibility) |

> **Silent failure alert — required deployment step:** Because the sidecar is non-essential, a crash leaves the task running with no telemetry collection. Deploy the EventBridge rule described in [alerting.md — A-01](alerting.md#a-01-ecs-sidecar-silent-telemetry-blackout) to detect tasks where `otelcol-contrib` has stopped but the app container remains running. Without this rule, telemetry blackouts for individual tasks are invisible to operators.

---

### Configuration

> **Endpoint configuration is currently undefined.** The following are placeholder values pending finalization of the Gateway NLB FQDN, TLS certificate distribution approach, and ECS task IAM role permissions.

Config is supplied to the sidecar at task startup via one of:
- **S3 config fetch** — startup command pulls `s3://infra-configs/otelcol/ecs-sidecar.yaml` on init.
- **SSM Parameter Store** — config rendered from a parameter and written to a temp file before the collector starts.
- **Baked into image** — only appropriate for stable, environment-agnostic base configs; env-specific values injected via `OTEL_*` environment variables in the task definition.

#### Receivers (sidecar build)

| Component | Purpose |
|-----------|---------|
| `otlpreceiver` | Receive traces, metrics, and logs from app container on `localhost:4317/4318` — all three signals share this single endpoint |
| `awsecscontainermetricsreceiver` | ECS task-level CPU, memory, network, storage metrics from the task metadata endpoint |

#### Processors

All six components from the [Common Processor Stack](#common-processor-stack). The `resourcedetectionprocessor` is configured to detect ECS task metadata (task ARN, cluster, region, AZ) rather than EC2 instance metadata.

#### Exporters

| Component | Purpose |
|-----------|------|
| `otlpexporter` | Forward all signals (gRPC, TLS) to Gateway NLB — **endpoint TBD** |

#### App Container Instrumentation Options

The sidecar is always present in every ECS task — it is the required transport layer. The choice of instrumentation approach is independent of the sidecar topology:

| Approach | Code changes | Best for |
|---|---|---|
| **Auto-instrumentation** (`opentelemetry-instrument` wrapper) | None — prefix the container start command | Standard containers where well-known libraries (requests, SQLAlchemy, boto3, etc.) are the main observability surface |
| **Manual OTel SDK** (`from opentelemetry import trace`, `metrics`, `logs`) | Yes — explicit SDK calls in application code | Services requiring custom spans, business-level metrics, or structured log correlation beyond auto-instrumented coverage |

Either way, the app emits to `localhost:4317` → sidecar → Gateway NLB. The sidecar config is identical regardless of which instrumentation approach the application uses.

---

## SDK & Instrumentation Patterns

Application-level instrumentation for environments where a standalone `otelcol-contrib` agent process is impractical (short-lived jobs, Lambda functions) or where SDK-level span and metric control is required. Each pattern routes telemetry to the Gateway NLB via TLS OTLP, either directly or through a co-located agent or sidecar.


---

### ETL Pipelines (Python)

#### Covered Systems

| System | OS | Signal Types |
|--------|----|--------------|
| Data/ETL Pipelines (Python) | Linux (AWS EC2) | Logs · Traces · Metrics |

#### How It Works

ETL pipelines run as Python processes on EC2 hosts where the [Standard Host Agent](#ec2--standard-host-agent-linux--windows) is already deployed. The **OTel Python SDK** emits traces, metrics, and logs to the local agent via `localhost:4317`; the agent applies the [common processor stack](#processing) and forwards all telemetry to the Gateway NLB via TLS OTLP. No additional agent process or infrastructure is required. Direct SDK instrumentation (explicit span boundaries per stage, custom business metrics) is preferred over the `opentelemetry-instrument` CLI wrapper for the control it provides over extract/transform/load stage boundaries.

#### SDK Configuration

All settings are supplied via environment variables (systemd unit env file or AWS SSM Parameter Store). The OTLP endpoint targets the **local agent**, not the Gateway NLB directly. TLS is enforced only on the agent → Gateway NLB leg.

| Variable | Value | Purpose |
|----------|-------|---------|
| `OTEL_SERVICE_NAME` | `etl-<pipeline-name>` | Identifies the pipeline in Tempo and dashboards |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4317` | Local host agent — no TLS needed on loopback |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` | Transport protocol |
| `OTEL_RESOURCE_ATTRIBUTES` | `deployment.environment=prod,team=data` | Static resource labels |
| `OTEL_TRACES_EXPORTER` | `otlp` | Enable trace export |
| `OTEL_METRICS_EXPORTER` | `otlp` | Enable metrics export |
| `OTEL_LOGS_EXPORTER` | `otlp` | Enable log export |
| `OTEL_PYTHON_LOG_CORRELATION` | `true` | Inject trace/span IDs into log records |

#### Required Python Packages

Installed into the pipeline's virtual environment or container image; pinned in the infrastructure repo alongside collector component versions.

| Package | Purpose |
|---------|---------|
| `opentelemetry-api` | Public API surface for explicit span, metric, and log calls in pipeline code |
| `opentelemetry-sdk` | Core SDK — TracerProvider, MeterProvider, LoggerProvider |
| `opentelemetry-exporter-otlp-proto-grpc` | OTLP gRPC exporter → local host agent on `localhost:4317` |

> Add library instrumentors (`opentelemetry-instrumentation-sqlalchemy`, `-psycopg2`, `-requests`, `-boto3sqs`) only if those libraries are present; call `.instrument()` manually at SDK init — **not** via the `opentelemetry-instrument` CLI wrapper.

#### Deployment Notes

- The Standard Host Agent must be deployed with `otlpreceiver` enabled on `localhost:4317` (included by default — no agent config changes needed).
- Set `OTEL_*` environment variables at deploy time via the systemd unit env file or AWS SSM-injected parameters.
- Initialize the OTel SDK at process startup, before any pipeline stage functions are called.

---

### Scheduled Jobs (Python)

#### Covered Systems

| System | Hosting | Signal Types |
|--------|---------|-------------|
| Scheduled Jobs | AWS (EC2, ECS, Lambda cron) | Logs · Traces · Metrics |
| Scheduled Jobs | DigitalOcean (Droplets, cron) | Logs · Traces · Metrics |

#### How It Works

The `opentelemetry-instrument` CLI entry point (shipped by the `opentelemetry-instrumentation` Python package) wraps script execution. The job invocation command is prefixed:

```
opentelemetry-instrument python my_job.py
```

At startup, the instrumentation layer:
1. Auto-patches any detected libraries in the script's imports (e.g., `requests`, `psycopg2`, `sqlalchemy`, `boto3`) to emit spans automatically.
2. Creates a root span for the script execution lifetime.
3. Captures stdout/stderr as log records.
4. Emits traces, metrics, and logs via OTLP to the configured endpoint.

The script itself requires **zero code changes**. The only modification is to the job invocation command and the addition of environment variables (see below).

#### Configuration via Environment Variables

All instrumentation settings are supplied through environment variables, set in the job's execution environment (systemd unit, cron wrapper script, ECS task definition env block, or DigitalOcean Droplet environment file):

| Variable | Value | Purpose |
|----------|-------|---------|
| `OTEL_SERVICE_NAME` | `job-name` (per job) | Identifies the job in traces and dashboards |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `https://nlb.company.com:443` | Gateway NLB OTLP endpoint |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` | Transport protocol |
| `OTEL_EXPORTER_OTLP_CERTIFICATE` | `/etc/otel/ca.crt` | TLS CA cert for NLB |
| `OTEL_RESOURCE_ATTRIBUTES` | `deployment.environment=prod,team=data` | Static resource labels |
| `OTEL_TRACES_EXPORTER` | `otlp` | Enable trace export |
| `OTEL_METRICS_EXPORTER` | `otlp` | Enable metrics export |
| `OTEL_LOGS_EXPORTER` | `otlp` | Enable log export |
| `OTEL_PYTHON_LOG_CORRELATION` | `true` | Inject trace/span IDs into log records |

#### Required Python Packages

Installed into the job's virtual environment or container image. Pinned versions are managed in the infrastructure repo and applied via the same DevOps pipeline used for other collector deployments.

| Package | Purpose |
|---------|---------|
| `opentelemetry-instrumentation` | Provides the `opentelemetry-instrument` CLI wrapper |
| `opentelemetry-sdk` | Core SDK (traces, metrics, logs) |
| `opentelemetry-exporter-otlp-proto-grpc` | OTLP gRPC exporter |
| `opentelemetry-instrumentation-requests` | Auto-instrument `requests` HTTP calls |
| `opentelemetry-instrumentation-sqlalchemy` | Auto-instrument SQLAlchemy DB queries |
| `opentelemetry-instrumentation-psycopg2` | Auto-instrument direct Postgres connections |
| `opentelemetry-instrumentation-boto3sqs` | Auto-instrument AWS SQS interactions |
| `opentelemetry-instrumentation-logging` | Bridge Python `logging` module into OTel logs |

> Add or remove instrumentation packages to match the libraries each job actually imports. Unused instrumentors add startup overhead.

#### Signal Collection Detail

**Traces**
- One root span per script execution; child spans created automatically for instrumented library calls (HTTP, DB, SQS).
- Span status set to `ERROR` on unhandled exceptions; exception details captured as span events.
- Sent to Grafana Tempo via Gateway NLB.

**Metrics**
- Runtime metrics (GC, memory, active threads) emitted by default via `opentelemetry-instrumentation-system-metrics` if included.
- Custom business metrics can be emitted via the OTel Python metrics API with no changes required to the wrapping approach.
- Sent to Prometheus via Gateway NLB (remote-write).

**Logs**
- Python `logging` module output bridged to OTel log records via `opentelemetry-instrumentation-logging`.
- Trace/span IDs automatically correlated into log records when `OTEL_PYTHON_LOG_CORRELATION=true`.
- Sent to Grafana Loki via Gateway NLB.

#### Deployment Notes

- **AWS jobs**: Update the task definition command (ECS) or systemd `ExecStart` / cron command to prefix with `opentelemetry-instrument`. Environment variables are injected via ECS task definition env block or Secrets Manager references.
- **DigitalOcean jobs**: Update the cron entry or systemd unit on the Droplet. Environment variables are set in a sourced env file (e.g., `/etc/otel/job.env`). The Droplet reaches the Gateway NLB over the site-to-site VPN.
- Package installation and env file management are handled by the same Ansible pipeline used for EC2 host agent deployments.
- For Lambda cron jobs, use the **OTel Lambda Layer** instead of this pattern (Lambda does not support persistent processes or `opentelemetry-instrument` wrapping in the same way).

---

### ADOT Lambda Layer

The **AWS Distro for OpenTelemetry (ADOT) Lambda Layer** is the primary collection method for AWS Lambda functions. It bundles `otelcol-contrib` and the OTel SDK into a managed Lambda Layer, automatically wrapping the function handler at runtime. No dependencies need to be added to the function's deployment package.

Once the layer is attached and the wrapper env var is set, the function gains full OTel instrumentation. Lambda code can additionally make direct SDK calls to emit custom spans, metrics, and logs alongside the automatic instrumentation.

---

#### Covered Systems

| System | Runtime | Signal Types |
|--------|---------|--------------|
| AWS Lambda functions | Python, Node.js, Java, .NET | Logs · Traces · Metrics |

---

#### How It Works

1. Attach the ADOT managed layer ARN to the Lambda function (region-specific; see [AWS ADOT Lambda docs](https://aws-otel.github.io/docs/getting-started/lambda)).
2. Set the environment variable `AWS_LAMBDA_EXEC_WRAPPER=/opt/otel-handler`. This tells the Lambda runtime to pass execution through the OTel wrapper before invoking the handler.
3. The wrapper starts an embedded `otelcol-contrib` process inside the execution environment, configured to export via OTLP to the Gateway NLB.
4. The handler executes normally. The wrapper creates a root span for the invocation and auto-instruments supported libraries (HTTP clients, AWS SDK calls, etc.).
5. On invocation completion, telemetry is flushed to the collector before the sandbox freezes.

##### Required Environment Variables

| Variable | Value |
|----------|-------|
| `AWS_LAMBDA_EXEC_WRAPPER` | `/opt/otel-handler` |
| `OPENTELEMETRY_COLLECTOR_CONFIG_FILE` | `/var/task/collector.yaml` (custom config; see below) |
| `OTEL_SERVICE_NAME` | Function name or logical service identifier |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `https://nlb.company.com:443` |
| `OTEL_RESOURCE_ATTRIBUTES` | `deployment.environment=prod,team=<team>` |
| `OTEL_PROPAGATORS` | `tracecontext,baggage` |

##### Collector Config (`collector.yaml` bundled in deployment package)

By default the ADOT layer exports to AWS X-Ray. Override with a custom config to route to the Gateway NLB instead:

```yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: "localhost:4317" }
exporters:
  otlp:
    endpoint: "nlb.company.com:443"
    tls: { insecure: false }
service:
  pipelines:
    traces:  { receivers: [otlp], exporters: [otlp] }
    metrics: { receivers: [otlp], exporters: [otlp] }
    logs:    { receivers: [otlp], exporters: [otlp] }
```

---

#### Sending Telemetry from Lambda Code

The layer provides initialised SDK globals. Functions can emit custom telemetry with minimal code.

**Trace — custom span**
```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def handler(event, context):
    with tracer.start_as_current_span("process-order") as span:
        span.set_attribute("order.id", event["order_id"])
        # ... business logic ...
```

**Metric — counter**
```python
from opentelemetry import metrics

meter = metrics.get_meter(__name__)
orders_counter = meter.create_counter("orders.processed", description="Orders completed")

def handler(event, context):
    # ... business logic ...
    orders_counter.add(1, {"status": "success"})
```

**Log — with automatic trace correlation**
```python
import logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def handler(event, context):
    logger.info("Order received", extra={"order.id": event["order_id"]})
    # trace_id and span_id are automatically injected into the log record
    # by the OTel logging bridge when OTEL_PYTHON_LOG_CORRELATION=true
```

> The root span for each Lambda invocation is created automatically by the wrapper. Custom spans added via the tracer become children of that root span with no extra setup.

---

#### Notes & Constraints

- **Cold start overhead**: The embedded collector adds ~200–500 ms on cold start. Acceptable for most workloads; avoid for latency-critical synchronous APIs where cold starts are frequent.
- **Flush on freeze**: The OTel SDK flushes telemetry before the sandbox freezes, but very short-lived functions should set `OTEL_BSP_MAX_EXPORT_BATCH_SIZE` and timeout values conservatively to avoid data loss.
- **Lambda logs → CloudWatch**: Lambda stdout/stderr always routes to CloudWatch Logs regardless of the layer. The Remote Collection Scraper (`awscloudwatchreceiver`) pulls these into Loki as a secondary log path.
- **Layer versioning**: Pin the ADOT layer version in Terraform / SAM / CDK. Layer updates are not automatic and should be tested before promotion.

---

### OTel SDK — Direct Instrumentation (opt-in)

An opt-in pattern for services where explicit SDK calls provide higher signal fidelity than the `opentelemetry-instrument` auto-instrumentation wrapper. Applies when manual span and metric control is warranted.

> **Fargate:** The OTel SDK is used by Fargate app containers to emit signals to the co-located sidecar at `localhost:4317`. Whether to use auto-instrumentation or manual SDK calls is a dev-side choice — both options always emit through the sidecar. See [App Container Instrumentation Options](#app-container-instrumentation-options) in the Sidecar section. This section does not affect Fargate topology.

> **Scheduled Jobs:** The authoritative pattern for Python scheduled jobs (AWS and DigitalOcean) is the `opentelemetry-instrument` wrapper documented in [Scheduled Jobs (Python)](#scheduled-jobs-python). Use the manual SDK approach only when the wrapper cannot provide the required span granularity or custom metric instruments.

#### When to Use

- Any service where `opentelemetry-instrument` auto-instrumentation coverage is insufficient for the required observability depth.
- Services requiring explicit custom spans, business-metric instruments, or structured log correlation beyond what auto-instrumented libraries expose.

#### Approach

The application imports the OTel SDK directly and configures an OTLP exporter pointed at the local host agent (`localhost:4317`). Where no host agent is running on the host, the exporter targets the Gateway NLB directly. Spans, metrics, and logs are emitted in-process with no additional collector process required.

SDK language packages follow the same OTLP-over-TLS transport used by all other collection methods. Endpoint and authentication settings are supplied via the standard `OTEL_*` environment variables, keeping application code environment-agnostic.

#### Trade-offs

| | OTel SDK (manual) | Auto-instrumentation wrapper |
|---|---|---|
| Code changes required | Yes — explicit SDK calls | None |
| Custom span / metric control | Full | Limited to auto-instrumented libraries |
| No additional process on host | Yes | Yes |
| Suitable for short-lived processes | Yes | Yes |

---

## Summary

| Instance Role | Deployment Count | OS | Signals | Ingest Direction |
|---------------|-----------------|-----|---------|-----------------|
| Standard EC2 Host Agent | One per EC2 host | Linux + Windows | Metrics · Logs · Traces | Push → Gateway NLB |
| ETL Pipelines (Python SDK) | One SDK per pipeline process; shares EC2 host agent | Linux (AWS EC2) | Metrics · Logs · Traces | SDK → Local Agent → Gateway NLB |
| Remote Collection Scraper | One dedicated instance | Linux | Metrics · Logs | Pull (API) → Push to Gateway NLB |
| Python Job Instrumentation | One per scheduled job process | Linux | Metrics · Logs · Traces | Push → Gateway NLB |
| OTel Sidecar | One per ECS task instance | Linux (container) | Metrics · Logs · Traces | Push → Gateway NLB |
| ADOT Lambda Layer | One layer per Lambda function | Lambda runtime | Metrics · Logs · Traces | Push → Gateway NLB |
| OTel SDK | Per application / service | Any | Metrics · Logs · Traces | Push → Gateway NLB |
| EKS DaemonSet Collector | One per EKS node | Linux (container) | Metrics · Logs · Traces | Push → Gateway NLB |
| EKS Pod Sidecar (optional) | One per isolated pod | Linux (container) | Metrics · Traces | Push → Gateway NLB |
