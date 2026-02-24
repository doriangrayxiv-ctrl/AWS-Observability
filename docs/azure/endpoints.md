# OTel Collection — Endpoint Architecture (Azure)

Defines how `otelcol-contrib` is deployed and configured across each collection scenario. All agents forward telemetry via TLS OTLP to the Gateway Azure Load Balancer at `gateway.company.com:443`. Agents perform **lightweight processing only** — coarse filtering, format normalisation, and batching — before data leaves the host. **Tail sampling is performed exclusively at the Gateway tier**, where the complete trace can be assembled from all contributing agents before a sampling decision is made.

> **Azure-specific notes:** This document replaces all AWS-specific infrastructure references (EC2, ECS, EKS, Lambda, SSM, ASG) with their Azure equivalents (Azure VMs, Azure Container Apps, AKS, Azure Functions, Azure Automation / Azure Arc, VMSS). The core OTel Collector configuration patterns — receivers, processors, exporters — remain identical. The `otelcol-contrib` binary is the same regardless of cloud platform.

---

## Common Components

Shared pipeline configuration applied across all `otelcol-contrib` agent and sidecar deployments. The processor ordering, enrichment rules, and TLS exporter settings defined here are applied by every agent and sidecar section below before telemetry is forwarded to the Gateway Azure LB at `gateway.company.com:443`.

---

### Processing

All agent deployments run the following processor stages in the collector pipeline before export. Processor configs are rendered from templates and deployed alongside receiver/exporter configs through Azure DevOps Pipelines using Terraform and Ansible.

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
- Scrubbing rules are maintained centrally in the pipeline config template and versioned in Azure Git Repos.

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
| `batchprocessor` | Batch spans/metrics/logs before export to reduce LB connections |
| `resourcedetectionprocessor` | Auto-attach cloud/host metadata (VM name, resource group, region, zone — or AKS node name / ACA revision depending on environment) |
| `attributesprocessor` | Attach static labels: `env`, `service.name`, `team` (from config template vars or container env vars) |
| `redactionprocessor` | PII scrubbing — block-listed attribute keys replaced with a redacted placeholder before export |
| `filterprocessor` | Drop debug logs (prod), unused metrics, health-check/readiness-probe spans, and duplicate records per environment rules |

---

## Agent Deployments

Persistent `otelcol-contrib` processes deployed on each host, node, or dedicated VM. All agent deployments apply the [Common Processor Stack](#common-processor-stack) and forward telemetry via TLS OTLP to the Gateway Azure LB. No tail sampling occurs at the agent — all spans are forwarded unsampled for Gateway-tier decisions.

---

### Azure VM — Standard Host Agent (Linux & Windows)

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

Configurations are **templated and deployed via Azure DevOps Pipelines** using Ansible for configuration management. A host-specific variable file supplies environment, role, and service identifiers that are injected into the collector config at deploy time.

For on-prem and DigitalOcean hosts, **Azure Arc** can be used to extend Azure management capabilities (VM extensions, policy, monitoring) to non-Azure machines, enabling consistent configuration distribution.

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
| `otlpexporter` | Forward all signals (gRPC, TLS) to `gateway.company.com:443` |

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

- Config templates live in the observability infrastructure repo in Azure Git Repos.
- Role-specific variable files (`vm-linux-db.vars`, `vm-windows-app.vars`, etc.) drive `SERVICE_NAME`, log paths, and which receivers are enabled.
- Azure DevOps Pipelines trigger Ansible playbooks that render the template and restart the collector service.
- Post-deploy health check hits `localhost:13133/health`; pipeline fails if non-200.
- Binary version is pinned in the pipeline; upgrades follow the same template render + restart flow.
- **Azure Arc-enabled servers** (on-prem and DigitalOcean) can receive config updates via Azure Automation Runbooks or Ansible executed from Azure DevOps.

---

### Azure VM — Remote Collection Scraper (Dedicated Instance)

A single dedicated Azure VM (`B2ms` or similar) running `otelcol-contrib` configured exclusively to pull telemetry from **external and managed services** that cannot host their own agent. This instance has no local application workload.

> This isolates external API credentials and polling schedules from general-purpose hosts and allows independent scaling and security controls.

#### Covered Systems

| System | Collection Method | Signal Types |
|--------|------------------|-------------|
| Shopify (Ecommerce SaaS) | Webhook receiver — Shopify pushes order/event data to `httplogreceiver` | Logs · Metrics |
| Retail POS (REST API) | `httplogreceiver` webhook inbound (if POS supports webhooks); Python cron script → `filelogreceiver` for pull-only APIs | Logs · Metrics |
| Azure Database for PostgreSQL/MySQL | `azuremonitorreceiver` (native Azure Monitor metrics + diagnostic logs) | Logs · Metrics |
| Snowflake (Data Warehouse) | `snowflakereceiver` (Snowflake REST API / information_schema queries) | Metrics |
| Azure Monitor | `azuremonitorreceiver` (pull metrics from Azure Monitor for any Azure resource) | Logs · Metrics |

#### Receiver Detail

##### Shopify
- Shopify does not expose a native OTel endpoint. The recommended path is a **Shopify webhook** configured to POST order/event payloads to an `httplogreceiver` listener on this instance.
- The `httplogreceiver` accepts inbound HTTP POST bodies and emits one log record per payload; no custom parsing code required in the collector.
- Fallback (if webhooks cannot be used): a Python script on cron calls the Shopify Admin REST API, appends newline-delimited JSON records to `/var/otel/shopify_events.ndjson`, and `filelogreceiver` tails that file.
- **Credential**: Shopify Webhook HMAC secret stored in Azure Key Vault; fetched at startup via Managed Identity.

##### Retail POS (REST API)
- If the POS system supports webhooks: configure the POS to push events to an `httplogreceiver` endpoint on this instance.
- If the POS is pull-only: a Python script on cron calls the POS REST API and appends newline-delimited JSON records to `/var/otel/pos_events.ndjson`; `filelogreceiver` tails that file and emits log records into the pipeline.
- Exact collection method depends on POS vendor capability — webhook preferred; cron poller as fallback.
- **Credential**: POS API key stored in Azure Key Vault.

##### Azure Database for PostgreSQL/MySQL
- Azure Database metrics are natively published to Azure Monitor. The `azuremonitorreceiver` pulls these on a configurable interval.
- Diagnostic logs (slow-query logs, error logs) are routed to Azure Monitor Logs or Event Hubs; pulled via the `azuremonitorreceiver` or `azureeventhubreceiver`.
- Managed Identity on this VM grants `Monitoring Reader` role scoped to the database resource group.
- No direct DB connectivity required from this instance for metrics. For detailed database-level metrics, the `postgresqlreceiver` / `mysqlreceiver` can connect directly using credentials from Key Vault.

##### Snowflake
- The `snowflakereceiver` (otelcol-contrib) queries Snowflake's `SNOWFLAKE.ACCOUNT_USAGE` schema for warehouse metrics, query history, storage, and credit usage.
- Requires a dedicated Snowflake service account with `MONITOR` privilege on the `SNOWFLAKE` database.
- **Credential**: Snowflake username/password or key-pair auth stored in Azure Key Vault.
- Poll interval: 300s (Snowflake latency for `ACCOUNT_USAGE` views is ~45 min; fine-grained real-time metrics are not available via this path).

##### Azure Monitor (General)
- Catch-all for any Azure services not covered by a dedicated receiver.
- The `azuremonitorreceiver` (otelcol-contrib) queries Azure Monitor REST APIs to pull metrics for configured Azure resource types.
- Configured with resource type and namespace filters (e.g., `Microsoft.Compute/virtualMachines`, `Microsoft.ContainerService/managedClusters`, `Microsoft.Web/sites`).
- Also used to pull Azure Diagnostic Logs from configured resources.
- Managed Identity: `Monitoring Reader` role scoped to the subscription or relevant resource groups.

#### Custom otelcol-contrib Build — Included Components

##### Receivers

| Component | Purpose |
|-----------|---------|
| `httplogreceiver` | Inbound webhook payloads (Shopify, POS) — emits one log record per HTTP POST body |
| `filelogreceiver` | Tails NDJSON output files written by Python cron pollers (POS pull-only, Shopify fallback) |
| `otlpreceiver` | Reserved for any future SDK-instrumented local processes |
| `azuremonitorreceiver` | Pull metrics + diagnostic logs from Azure Monitor (Azure DB, Azure Functions, general Azure resources) |
| `snowflakereceiver` | Pull Snowflake account usage metrics |

##### Processors

All six components from the [Common Processor Stack](#common-processor-stack). The `attributesprocessor` additionally attaches a `source.system` label (e.g., `shopify`, `azure-db`, `snowflake`) via config template vars to identify the originating system on every record.

##### Exporters

| Component | Purpose |
|-----------|------|
| `otlpexporter` | Forward all signals (gRPC, TLS) to `gateway.company.com:443` |

#### Security & Credentials

- Managed Identity on this VM grants scoped Azure Monitor read permissions — no long-lived credentials needed for Azure resources.
- All other secrets (Snowflake, POS API key, Shopify HMAC) are fetched at startup from **Azure Key Vault** via Managed Identity.
- NSG restricts inbound `httplogreceiver` webhook ports to known source IP ranges (Shopify IP ranges, POS datacenter IPs). Specific port assignments are defined during detailed config — out of scope for this high-level design.
- No inbound SSH; use Azure Bastion for access.

---

### AKS DaemonSet

The primary collection method for Azure Kubernetes Service (AKS) workloads. `otelcol-contrib` is deployed as a Kubernetes **DaemonSet**, placing one collector pod on every node in the cluster. This mirrors the Azure VM host agent pattern at the node level and provides automatic coverage for all pods running on a node without requiring per-pod configuration changes.

An optional **per-pod sidecar** supplements the DaemonSet for workloads requiring pod-level processing isolation. It is not the default and should not be deployed cluster-wide.

---

#### Covered Systems

| System | Hosting | Signal Types |
|--------|---------|-------------|
| Application pods | AKS (all node pool types) | Logs · Traces · Metrics |
| Kubernetes cluster state | AKS | Metrics |

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
┌─── AKS Node ──────────────────────────────────────────────────────────┐
│                                                                        │
│  ┌─── Pod A ────────────────┐    ┌─── Pod B (isolated) ─────────────┐ │
│  │  App Container           │    │  App Container                   │ │
│  │  (OTel SDK / auto-instr) │    │  (OTel SDK / auto-instr)         │ │
│  │                          │    │  ┌───────────────────────────┐   │ │
│  │  Traces/Metrics → OTLP   │    │  │  otelcol-contrib Sidecar  │   │ │
│  │  Logs    → stdout        │    │  │  (optional, per-pod)      │   │ │
│  └──────────┬───────────────┘    │  │  receives on localhost:   │   │ │
│             │ OTLP               │  │  4317, forwards to        │   │ │
│             │ hostIP:4317        │  │  DaemonSet or Gateway LB  │   │ │
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
│              │  resourcedetectionprocessor (Azure VM metadata)    │   │
│              │  attributesprocessor     (env, team, cluster.name) │   │
│              │  redactionprocessor      (PII scrub)               │   │
│              │  filterprocessor         (debug logs, healthchecks)│   │
│              │  memorylimiterprocessor + batchprocessor           │   │
│              └───────────────────────┬────────────────────────────┘   │
└──────────────────────────────────────┼────────────────────────────────┘
                                       │ TLS OTLP gRPC
                                       ▼
                           gateway.company.com:443
                          (Azure LB → Gateway VMSS)
                                       │
               ┌───────────────────────┼────────────────────┐
               ▼                       ▼                     ▼
          Prometheus                  Loki                 Tempo
          (metrics)                  (logs)               (traces)
               └───────────────────────┴─────────────────────┘
                                       ▼
                           grafana.company.com:443
                   (Front Door WAF → Azure Managed Grafana)
```

---

#### Signal Flow Detail

##### Metrics

| Source | Receiver | What Is Captured |
|--------|----------|-----------------|
| Node OS | `hostmetricsreceiver` | CPU, memory, disk I/O, network I/O — same scrape set as Azure VM host agent |
| Kubelet stats | `kubeletstatsreceiver` | Per-pod and per-container CPU, memory, network, filesystem — pulled from the Kubelet `/stats/summary` endpoint on each node |
| Cluster state | `k8sclusterreceiver` | Deployment replica counts, pod phase counts, node conditions, HPA state — **runs on one DaemonSet pod only**, elected via a `k8sleaderelector` extension to prevent duplicate cluster-level metrics |
| App OTel SDK | `otlpreceiver` | Custom business and runtime metrics emitted by instrumented pods; received on `hostIP:4317` |

All metrics → `DaemonSet → Gateway LB → Prometheus remote-write`.

##### Logs

| Source | Receiver | What Is Captured |
|--------|----------|-----------------|
| Pod stdout/stderr | `filelogreceiver` | Tails `/var/log/pods/*/*/*.log` mounted from the node's filesystem via `hostPath`; covers all containers on the node automatically |
| Node system logs | `filelogreceiver` | `/var/log/messages` or `/var/log/syslog` depending on node OS |

The `filelogreceiver` operators parse the Kubernetes container log wrapper format (JSON with `log`, `time`, `stream` fields) to extract the inner log body. `k8sattributesprocessor` enriches every log record with `k8s.namespace.name`, `k8s.pod.name`, `k8s.container.name`, and `k8s.deployment.name` by correlating the file path against the Kubernetes API.

All logs → `DaemonSet → Gateway LB → Grafana Loki`.

##### Traces

| Source | Receiver | What Is Captured |
|--------|----------|-----------------|
| App pods (SDK / auto-instr) | `otlpreceiver` | Spans pushed from app containers to the node's host IP on port 4317; pod resolves the address via the Kubernetes Downward API (see below) |
| Optional sidecar | `otlpreceiver` | For isolated pods: sidecar receives on `localhost:4317`, applies pod-level processing, then forwards to the DaemonSet or directly to the Gateway LB |

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

All traces → `DaemonSet → Gateway LB → Grafana Tempo`.

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

All six components from the [Common Processor Stack](#common-processor-stack), plus one AKS-specific addition:

| Addition | Purpose |
|----------|--------|
| `k8sattributesprocessor` | Enrich spans and log records with pod, namespace, and deployment metadata via K8s API |

The `resourcedetectionprocessor` is configured for Azure VM metadata (VM name, resource group, region, zone) and Kubernetes node name.

##### Exporters

| Component | Purpose |
|-----------|------|
| `otlpexporter` | Forward all signals (gRPC, TLS) to `gateway.company.com:443` |

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
- **Config management**: Config stored in a `ConfigMap` and mounted into the DaemonSet pod. Updates are applied via `kubectl rollout restart daemonset/otelcol-contrib` triggered from Azure DevOps Pipelines.
- **TLS CA cert**: Mount the Gateway TLS CA certificate into the pod from a Kubernetes `Secret` (sourced from Azure Key Vault via `csi-secrets-store-provider-azure` or `external-secrets` operator) and reference it in the `otlpexporter` TLS stanza.
- **Tolerations**: Add tolerations for control-plane node taints if node-level metrics from control plane nodes are required.
- **Node selector**: Scope to Linux nodes only (`kubernetes.io/os: linux`) if Windows node pools are present in the cluster.

---

## Sidecar Deployments

The primary collection method for containerized workloads running on **Azure Container Apps (ACA)**. An `otelcol-contrib` container is co-deployed alongside each application container within the same container app, sharing the container app's network namespace. This is the standard agent pattern for ACA, where no host OS access exists.

---

### How Many Sidecars?

**One sidecar container per container app** — not one per environment or per resource group.

Because ACA containers are isolated at the network level, a single centralized collector cannot reach other container apps' localhost interfaces. Each container app must carry its own collector sidecar to capture that app's OTLP output. This means sidecar count scales linearly with running container app count, which is expected and manageable: the sidecar is lightweight (128–256 MB RAM, minimal CPU at idle) and its resource allocation is defined in the container app alongside the app container.

---

### Covered Systems

| System | Hosting | Signal Types |
|--------|---------|-------------|
| Application containers | Azure Container Apps | Logs · Traces · Metrics |

---

### Sidecar Architecture

Each container app is updated to include `otelcol-contrib` as a sidecar container. The two containers share a network namespace, so the app container emits OTLP to `localhost:4317` / `localhost:4318` and the sidecar receives it without any cross-app or cross-host networking.

```
┌─── Azure Container App ───────────────────────────────────┐
│                                                             │
│  ┌─────────────────────┐   OTLP (localhost:4317)           │
│  │   App Container     │ ─────────────────────────►        │
│  │  (your service)     │   traces · metrics · logs         │
│  │                     │   (via OTel SDK imports)          │
│  └─────────────────────┘                                   │
│                                                             │
│  ┌─────────────────────┐                                   │
│  │  otelcol-contrib    │ ──► TLS OTLP ──► Gateway LB      │
│  │  Sidecar            │     gateway.company.com:443       │
│  └─────────────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

The sidecar:
- Receives **traces, metrics, and logs** from the app container via `otlpreceiver` on `localhost:4317/4318`. All three signals share the same OTLP transport.
- Applies the standard processor stack (resource detection, attribute enrichment, PII scrubbing, batching).
- Exports all signals via TLS OTLP to the Gateway Azure LB.

> **Azure Container Apps sidecar behaviour:** Unlike ECS Fargate where sidecars can be marked `essential: false` (sidecar crash doesn't affect the app), ACA sidecar containers follow the container app's restart policy. If the sidecar crashes, the container app's restart policy (`Always`, `OnFailure`, or `Never`) determines whether it restarts. With `OnFailure` or `Always` (recommended), the sidecar restarts automatically without stopping the app container. This means brief telemetry gaps during sidecar restarts rather than permanent silent blackouts. The failure model differs from ECS: ACA self-heals the sidecar; ECS requires task replacement.

> **Container logs:** ACA captures container stdout/stderr to Azure Monitor Logs (Log Analytics workspace) automatically via the platform's built-in log driver. This is independent of the sidecar and not collected by it. The sidecar receives logs exclusively via OTLP from the OTel SDK.

---

### Sidecar Container Requirements

| Setting | Value |
|---------|-------|
| Image | `otel/opentelemetry-collector-contrib:<pinned-version>` from Azure Container Registry |
| CPU allocation | 0.25 vCPU |
| Memory allocation | 0.5 Gi |
| Restart policy | `OnFailure` — auto-restart on crash; app container unaffected |
| Config | Azure Key Vault reference or baked-in config via env vars |
| Diagnostics | Container stdout → Azure Monitor Logs for operator visibility |

---

### Configuration

Config is supplied to the sidecar at container app startup via one of:
- **Azure Key Vault reference** — config file stored as a Key Vault secret, mounted as a volume in the container app.
- **Environment variables** — `OTEL_*` variables set in the container app configuration for base settings; detailed config baked into the container image.
- **Azure App Configuration** — centralised config store; container app references config values at startup.

#### Receivers (sidecar build)

| Component | Purpose |
|-----------|---------|
| `otlpreceiver` | Receive traces, metrics, and logs from app container on `localhost:4317/4318` — all three signals share this single endpoint |

> **Note:** There is no direct equivalent of the AWS `awsecscontainermetricsreceiver` for ACA. Container-level CPU, memory, and network metrics for ACA are available via Azure Monitor (pulled by the Remote Collection Scraper's `azuremonitorreceiver`). The sidecar focuses on application-level OTLP signals.

#### Processors

All six components from the [Common Processor Stack](#common-processor-stack). The `resourcedetectionprocessor` is configured to detect Azure metadata (subscription, resource group, region) and ACA container app metadata where available.

#### Exporters

| Component | Purpose |
|-----------|------|
| `otlpexporter` | Forward all signals (gRPC, TLS) to Gateway Azure LB |

#### App Container Instrumentation Options

The sidecar is always present in every container app — it is the required transport layer. The choice of instrumentation approach is independent of the sidecar topology:

| Approach | Code changes | Best for |
|---|---|---|
| **Auto-instrumentation** (`opentelemetry-instrument` wrapper) | None — prefix the container start command | Standard containers where well-known libraries (requests, SQLAlchemy, boto3, etc.) are the main observability surface |
| **Manual OTel SDK** (`from opentelemetry import trace`, `metrics`, `logs`) | Yes — explicit SDK calls in application code | Services requiring custom spans, business-level metrics, or structured log correlation beyond auto-instrumented coverage |

Either way, the app emits to `localhost:4317` → sidecar → Gateway Azure LB. The sidecar config is identical regardless of which instrumentation approach the application uses.

---

## SDK & Instrumentation Patterns

Application-level instrumentation for environments where a standalone `otelcol-contrib` agent process is impractical (short-lived jobs, Azure Functions) or where SDK-level span and metric control is required. Each pattern routes telemetry to the Gateway Azure LB via TLS OTLP, either directly or through a co-located agent or sidecar.

---

### ETL Pipelines (Python)

#### Covered Systems

| System | OS | Signal Types |
|--------|----|--------------|
| Data/ETL Pipelines (Python) | Linux (Azure VM) | Logs · Traces · Metrics |

#### How It Works

ETL pipelines run as Python processes on Azure VMs where the [Standard Host Agent](#azure-vm--standard-host-agent-linux--windows) is already deployed. The **OTel Python SDK** emits traces, metrics, and logs to the local agent via `localhost:4317`; the agent applies the [common processor stack](#processing) and forwards all telemetry to the Gateway Azure LB via TLS OTLP. No additional agent process or infrastructure is required. Direct SDK instrumentation (explicit span boundaries per stage, custom business metrics) is preferred over the `opentelemetry-instrument` CLI wrapper for the control it provides over extract/transform/load stage boundaries.

#### SDK Configuration

All settings are supplied via environment variables (systemd unit env file or Azure App Configuration). The OTLP endpoint targets the **local agent**, not the Gateway Azure LB directly. TLS is enforced only on the agent → Gateway LB leg.

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

> Add library instrumentors (`opentelemetry-instrumentation-sqlalchemy`, `-psycopg2`, `-requests`) only if those libraries are present; call `.instrument()` manually at SDK init — **not** via the `opentelemetry-instrument` CLI wrapper.

#### Deployment Notes

- The Standard Host Agent must be deployed with `otlpreceiver` enabled on `localhost:4317` (included by default — no agent config changes needed).
- Set `OTEL_*` environment variables at deploy time via the systemd unit env file or Azure App Configuration references.
- Initialize the OTel SDK at process startup, before any pipeline stage functions are called.

---

### Scheduled Jobs (Python)

#### Covered Systems

| System | Hosting | Signal Types |
|--------|---------|-------------|
| Scheduled Jobs | Azure (VMs, ACA Jobs, Azure Functions timer trigger) | Logs · Traces · Metrics |
| Scheduled Jobs | DigitalOcean (Droplets, cron) | Logs · Traces · Metrics |

#### How It Works

The `opentelemetry-instrument` CLI entry point (shipped by the `opentelemetry-instrumentation` Python package) wraps script execution. The job invocation command is prefixed:

```
opentelemetry-instrument python my_job.py
```

At startup, the instrumentation layer:
1. Auto-patches any detected libraries in the script's imports (e.g., `requests`, `psycopg2`, `sqlalchemy`) to emit spans automatically.
2. Creates a root span for the script execution lifetime.
3. Captures stdout/stderr as log records.
4. Emits traces, metrics, and logs via OTLP to the configured endpoint.

The script itself requires **zero code changes**. The only modification is to the job invocation command and the addition of environment variables (see below).

#### Configuration via Environment Variables

All instrumentation settings are supplied through environment variables, set in the job's execution environment (systemd unit, cron wrapper script, ACA Job container definition, or DigitalOcean Droplet environment file):

| Variable | Value | Purpose |
|----------|-------|---------|
| `OTEL_SERVICE_NAME` | `job-name` (per job) | Identifies the job in traces and dashboards |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `https://gateway.company.com:443` | Gateway Azure LB OTLP endpoint |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` | Transport protocol |
| `OTEL_EXPORTER_OTLP_CERTIFICATE` | `/etc/otel/ca.crt` | TLS CA cert for Gateway |
| `OTEL_RESOURCE_ATTRIBUTES` | `deployment.environment=prod,team=data` | Static resource labels |
| `OTEL_TRACES_EXPORTER` | `otlp` | Enable trace export |
| `OTEL_METRICS_EXPORTER` | `otlp` | Enable metrics export |
| `OTEL_LOGS_EXPORTER` | `otlp` | Enable log export |
| `OTEL_PYTHON_LOG_CORRELATION` | `true` | Inject trace/span IDs into log records |

#### Required Python Packages

| Package | Purpose |
|---------|---------|
| `opentelemetry-instrumentation` | Provides the `opentelemetry-instrument` CLI wrapper |
| `opentelemetry-sdk` | Core SDK (traces, metrics, logs) |
| `opentelemetry-exporter-otlp-proto-grpc` | OTLP gRPC exporter |
| `opentelemetry-instrumentation-requests` | Auto-instrument `requests` HTTP calls |
| `opentelemetry-instrumentation-sqlalchemy` | Auto-instrument SQLAlchemy DB queries |
| `opentelemetry-instrumentation-psycopg2` | Auto-instrument direct Postgres connections |
| `opentelemetry-instrumentation-logging` | Bridge Python `logging` module into OTel logs |

> Add or remove instrumentation packages to match the libraries each job actually imports. Unused instrumentors add startup overhead.

#### Deployment Notes

- **Azure VMs**: Update the systemd `ExecStart` or cron command to prefix with `opentelemetry-instrument`. Environment variables injected via systemd env file.
- **ACA Jobs**: Configure the container command to use `opentelemetry-instrument` as the entrypoint. Environment variables set in the container app job configuration.
- **DigitalOcean jobs**: Update the cron entry or systemd unit on the Droplet. Environment variables are set in a sourced env file (e.g., `/etc/otel/job.env`). The Droplet reaches the Gateway Azure LB over the site-to-site VPN.
- Package installation and env file management are handled by Azure DevOps Pipelines using Ansible.

---

### Azure Functions — OTel SDK Instrumentation

Azure Functions is the serverless compute platform equivalent to AWS Lambda. Unlike AWS which offers the managed ADOT Lambda Layer, Azure Functions requires **direct OTel SDK integration** in the function code. There is no managed OTel layer for Azure Functions — the SDK is added as a dependency to the function's project.

---

#### Covered Systems

| System | Runtime | Signal Types |
|--------|---------|--------------|
| Azure Functions | Python, Node.js, Java, .NET | Logs · Traces · Metrics |

---

#### How It Works

1. Add the OTel SDK packages to the function's project dependencies (`requirements.txt` for Python, `package.json` for Node.js).
2. Initialize the OTel SDK at function app startup (e.g., in `__init__.py` or a startup hook).
3. Configure the OTLP exporter to point to the Gateway Azure LB at `gateway.company.com:443`.
4. The SDK automatically creates spans for each function invocation and auto-instruments supported libraries.
5. On invocation completion, telemetry is flushed before the function instance idles.

> **Key difference from AWS:** AWS provides the ADOT Lambda Layer, a managed wrapper that bundles an embedded `otelcol-contrib` process. Azure Functions has no equivalent managed layer. The OTel SDK runs in-process within the function and exports directly to the Gateway Azure LB — no embedded collector is needed.

##### Required Environment Variables (Application Settings)

| Variable | Value |
|----------|-------|
| `OTEL_SERVICE_NAME` | Function app name or logical service identifier |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `https://gateway.company.com:443` |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` |
| `OTEL_RESOURCE_ATTRIBUTES` | `deployment.environment=prod,team=<team>` |
| `OTEL_PROPAGATORS` | `tracecontext,baggage` |

##### Python Example — Function App with OTel SDK

```python
# function_app.py
import azure.functions as func
import logging
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

# Initialize OTel SDK at module level (shared across invocations)
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter())
)
metrics.set_meter_provider(MeterProvider(
    metric_readers=[PeriodicExportingMetricReader(OTLPMetricExporter())]
))

tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)
orders_counter = meter.create_counter("orders.processed")

app = func.FunctionApp()

@app.function_name(name="ProcessOrder")
@app.route(route="orders")
def process_order(req: func.HttpRequest) -> func.HttpResponse:
    with tracer.start_as_current_span("process-order") as span:
        order_id = req.params.get("order_id")
        span.set_attribute("order.id", order_id)
        # ... business logic ...
        orders_counter.add(1, {"status": "success"})
        return func.HttpResponse("OK", status_code=200)
```

#### Required Python Packages (`requirements.txt`)

| Package | Purpose |
|---------|---------|
| `opentelemetry-api` | Public API surface |
| `opentelemetry-sdk` | Core SDK (traces, metrics, logs) |
| `opentelemetry-exporter-otlp-proto-grpc` | OTLP gRPC exporter → Gateway Azure LB |
| `opentelemetry-instrumentation-requests` | Auto-instrument HTTP calls (if used) |
| `opentelemetry-instrumentation-logging` | Bridge Python `logging` into OTel logs |

---

#### Notes & Constraints

- **Cold start overhead**: OTel SDK initialization adds ~100–300 ms on cold start. Less than the ADOT Lambda Layer (~200–500 ms) because there is no embedded collector process to start.
- **Flush on idle**: Configure `BatchSpanProcessor` with conservative `schedule_delay_millis` and `max_export_batch_size` to ensure telemetry is flushed before the function instance is frozen/recycled.
- **Function logs → Azure Monitor**: Azure Functions stdout/stderr always routes to Azure Monitor Logs regardless of OTel SDK usage. The Remote Collection Scraper (`azuremonitorreceiver`) can pull these as a secondary log path if needed.
- **SDK version pinning**: Pin OTel SDK versions in `requirements.txt` / `package.json`. Updates are tested in staging before production promotion via Azure DevOps Pipelines.

---

### OTel SDK — Direct Instrumentation (opt-in)

An opt-in pattern for services where explicit SDK calls provide higher signal fidelity than the `opentelemetry-instrument` auto-instrumentation wrapper. Applies when manual span and metric control is warranted.

> **ACA:** The OTel SDK is used by ACA app containers to emit signals to the co-located sidecar at `localhost:4317`. Whether to use auto-instrumentation or manual SDK calls is a dev-side choice — both options always emit through the sidecar.

> **Scheduled Jobs:** The authoritative pattern for Python scheduled jobs (Azure and DigitalOcean) is the `opentelemetry-instrument` wrapper documented in [Scheduled Jobs (Python)](#scheduled-jobs-python). Use the manual SDK approach only when the wrapper cannot provide the required span granularity or custom metric instruments.

#### When to Use

- Any service where `opentelemetry-instrument` auto-instrumentation coverage is insufficient for the required observability depth.
- Services requiring explicit custom spans, business-metric instruments, or structured log correlation beyond what auto-instrumented libraries expose.

#### Approach

The application imports the OTel SDK directly and configures an OTLP exporter pointed at the local host agent (`localhost:4317`). Where no host agent is running on the host, the exporter targets the Gateway Azure LB directly. Spans, metrics, and logs are emitted in-process with no additional collector process required.

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
| Standard Azure VM Host Agent | One per Azure VM | Linux + Windows | Metrics · Logs · Traces | Push → Gateway Azure LB |
| ETL Pipelines (Python SDK) | One SDK per pipeline process; shares VM host agent | Linux (Azure VM) | Metrics · Logs · Traces | SDK → Local Agent → Gateway Azure LB |
| Remote Collection Scraper | One dedicated VM | Linux | Metrics · Logs | Pull (API) → Push to Gateway Azure LB |
| Python Job Instrumentation | One per scheduled job process | Linux | Metrics · Logs · Traces | Push → Gateway Azure LB |
| OTel Sidecar | One per ACA container app | Linux (container) | Metrics · Logs · Traces | Push → Gateway Azure LB |
| Azure Functions (OTel SDK) | One SDK per function app | Functions runtime | Metrics · Logs · Traces | Push → Gateway Azure LB |
| OTel SDK | Per application / service | Any | Metrics · Logs · Traces | Push → Gateway Azure LB |
| AKS DaemonSet Collector | One per AKS node | Linux (container) | Metrics · Logs · Traces | Push → Gateway Azure LB |
| AKS Pod Sidecar (optional) | One per isolated pod | Linux (container) | Metrics · Traces | Push → Gateway Azure LB |
