# Futures — Deferred Features (Azure)

Features and design improvements that are acknowledged as important but are **out of scope for the current high-level functional design**. Revisit during production hardening or a dedicated follow-on phase.

---

## F1. Backpressure / Persistent Queue Strategy

- **Origin:** arch_eval.md P1 #5
- **Scope decision:** For the current design, telemetry loss on Gateway failure is accepted as a known limitation.
- **Context:** If the Gateway Azure Load Balancer or backends are unreachable, the agent-side `batchprocessor` will eventually drop data. No `file_storage` extension is specified for persistent queuing on agents/sidecars, and no dead-letter path exists.
- **Why it matters (when revisited):** During incidents — exactly when observability is most critical — telemetry is lost without a buffer. This is particularly severe for short-lived workloads (scheduled jobs, Azure Functions) where in-memory queues are discarded on process exit.
- **Future implementation path:**
  - Enable the `file_storage` extension on all agent and sidecar OTel Collector configs.
  - Configure the `otlpexporter` sending queue to use `file_storage` as its persistent backend.
  - Specify a per-host disk path (e.g., `/var/otel/queue`) and a max size cap to prevent disk exhaustion.
  - Add `file_storage` to each `ocb` (OTel Collector Builder) build's extensions list.
  - For DigitalOcean scheduled jobs, deploy a local `otelcol-contrib` agent per Droplet so job SDK output targets `localhost:4317`; the agent buffers and retries to the Gateway Azure LB.
  - For Azure Functions, persistent queuing is impractical (ephemeral compute). Consider an intermediate buffer (e.g., Azure Event Hub as a dead-letter sink) for Functions telemetry if direct-to-Gateway export failures are observed.

---

## F2. Gateway Collector — Detailed Architecture

- **Origin:** arch_eval.md P1 #6
- **Scope decision:** For the current design, the Gateway is assumed to handle core filtering, processing, enrichment, transformation, and routing of all telemetry to the Grafana backend stack (Prometheus, Loki, Tempo). Internal config details are deferred.
- **Context:** Every agent section references forwarding to the Gateway Azure LB, but no dedicated Gateway Collector config, scaling policy, or operational runbook exists.
- **Why it matters (when revisited):** The Gateway is the most critical single component in the pipeline. It is the sole point where tail sampling, PII scrubbing, cross-signal routing, and backend fan-out occur. Without a documented config it cannot be built, reviewed, or operated safely.
- **Future implementation path:**
  - Create a dedicated `gateway.md` (or a Gateway section in `endpoints.md`) covering:
    - **Receivers:** `otlpreceiver` on gRPC `:4317` and HTTP `:4318` with TLS termination (certificates loaded from Azure Key Vault via Managed Identity).
    - **Processors:** `tailsamplingprocessor` (trace decisions requiring full trace assembly), `filterprocessor` (drop noisy/low-value signals), `redactionprocessor` (PII scrubbing), `attributesprocessor` (label normalization), `resourceprocessor` (environment enrichment), `memorylimiterprocessor`, `batchprocessor`.
    - **Exporters:** `prometheusremotewriteexporter` → Prometheus, `lokiexporter` (or `otlpexporter` to Loki) → Loki, `otlpexporter` → Grafana Tempo.
    - **Routing:** `routingprocessor` or pipeline fan-out to direct metrics/logs/traces to the correct exporter.
    - **VMSS scaling policy:** Autoscale rules based on CPU utilisation and OTel Collector queue depth (custom metrics published to Azure Monitor or scraped by Prometheus); pre-scale before incident conditions degrade capacity.
    - **Health monitoring:** Gateway internal metrics (`/metrics` scrape endpoint) exported to Prometheus using `azure_sd_configs` for automatic instance discovery; dashboard in Grafana for queue depth, export errors, spans/metrics/logs received and exported, and VMSS instance count.
  - Add cross-references from each agent section to the Gateway section.

---

## F3. TLS Certificate Distribution and Rotation

- **Origin:** arch_eval.md P1 #9
- **Scope decision:** Certificate provisioning and distribution are out of scope for the current high-level design. All agent configs reference a CA cert path (e.g., `/etc/otel/ca.crt`, Kubernetes Secret, ACA sidecar env) but the sourcing and lifecycle management are deferred.
- **Context:** All agent-to-Gateway OTLP transport uses TLS. The CA cert must be provisioned on every host type (Azure VM, AKS nodes, ACA sidecars, DigitalOcean Droplets, Azure Functions) and rotated before expiry. No strategy currently exists for this.
- **Why it matters (when revisited):** An expired or missing CA cert causes a complete telemetry blackout across all agents — precisely the worst time to lose observability visibility.
- **Future implementation path:**
  - Select a CA source: **Azure Key Vault certificates** (recommended for Azure-native hosts), or a self-signed CA stored in Azure Key Vault as a secret.
  - **Azure VMs (Linux/Windows):** Distribute via Ansible playbooks triggered by Azure DevOps Pipelines; reload agent on rotation via `systemd` drop-in (Linux) or scheduled task restart (Windows). For Azure Arc-enabled servers, leverage the Key Vault VM extension to auto-fetch and rotate certificates.
  - **AKS:** Store as a Kubernetes Secret; mount into DaemonSet pods; rotate via `cert-manager` with an Azure Key Vault issuer, or the `external-secrets` operator syncing from Key Vault.
  - **Azure Container Apps:** Inject the cert as a secret reference in the container app configuration; rotate by updating the secret and deploying a new revision. ACA supports Key Vault secret references natively.
  - **DigitalOcean:** Copy cert via Ansible or cloud-init on Droplet provisioning; use a cron-based rotation script that signals the agent. For Azure Arc-enrolled DO hosts, the Key Vault VM extension can manage rotation.
  - **Azure Functions:** Store the CA cert as an Azure Key Vault reference in Function App settings; the SDK reads it at startup. Rotate by updating the Key Vault secret; the Function App picks up the new value on next cold start or via a configuration reload trigger.
  - Define a rotation runbook: alert 30 days before expiry (Azure Monitor alert on Key Vault certificate expiry event), test connectivity after rotation, confirm via Gateway export-success metrics.

---

## F4. Telemetry Buffering for DigitalOcean Scheduled Jobs

- **Origin:** arch_eval.md P1 #10
- **Scope decision:** Telemetry loss on VPN or Azure LB disruption is accepted as a known limitation for the current design. Low-cost direct export (OTel Python SDK → `gateway.company.com:443`) is the chosen pattern. A local buffering agent will be considered as the platform matures.
- **Context:** DigitalOcean scheduled jobs are short-lived processes that export telemetry directly to the Gateway Azure LB via WireGuard/VPN. If the tunnel or LB is momentarily unreachable during a job run, all telemetry for that run is silently lost. In-memory SDK queues are discarded on process exit.
- **Why it matters (when revisited):** Scheduled jobs often execute critical business logic (billing reconciliation, inventory sync). Silent loss of traces and metrics for these jobs obscures errors and performance regressions in high-value workflows.
- **Future implementation path:**
  - Deploy a persistent `otelcol-contrib` agent on each DigitalOcean Droplet that hosts scheduled jobs.
  - Configure the agent with the `file_storage` extension to back the `otlpexporter` sending queue with on-disk buffering (e.g., `/var/otel/queue`, max 1 GB).
  - Point all job SDK exporters at `localhost:4317`; the agent absorbs backpressure and retries to `gateway.company.com:443` when the VPN recovers.
  - This improvement is contingent on better understanding of DigitalOcean job frequency, duration, and telemetry volume to justify the added Droplet resource usage.

---

## F5. Capacity Planning and Storage Sizing

- **Origin:** arch_eval.md P1 #13
- **Scope decision:** Actual ingest volumes are unknown until the platform is operational and instrumentation coverage is complete. For the current design, total ingest is scoped to **8 GB/day** across all signals with **90-day retention**. A validated capacity model will be developed once real traffic data is available.
- **Current assumptions (design-level estimates):**

  | Signal | Daily ingest (raw) | Compression ratio | Daily stored | 90-day stored |
  |--------|--------------------|------------------|-------------|---------------|
  | Metrics | ~3 GB | 3:1 | ~1.0 GB | ~90 GB |
  | Logs | ~4 GB | 10:1 | ~0.4 GB | ~36 GB |
  | Traces | ~1 GB | 5:1 | ~0.2 GB | ~18 GB |
  | **Total** | **~8 GB** | | **~1.6 GB/day** | **~144 GB** |

  > These figures are rough order-of-magnitude estimates. Real compression will vary by signal density, cardinality, and schema. Managed Disk and VM sizing in `presentation.md` are based on these assumptions; revise when measured data is available.

- **Context:** `presentation.md` uses a hybrid storage model: ~180 GB Azure Managed Disks (Prometheus TSDB + Loki/Tempo WAL and index) plus ~54 GB in Azure Blob Storage (Loki chunks + Tempo blocks), which provides headroom above the 144 GB compressed estimate at lower cost than all-disk. Gateway VMSS instance count (2–4× B2ms) is not formally derived from ingest throughput at this stage.
- **Why it matters (when revisited):** Under-provisioned storage causes silent data loss when backends hit retention limits or disk exhaustion. Over-provisioned Gateway capacity wastes compute budget. Neither failure mode is acceptable in production.
- **Future implementation path:**
  - After 2–4 weeks of live operation, measure actual ingest rates per signal type from Prometheus (`otelcol_receiver_accepted_*`) and backend disk usage.
  - Build a capacity model: `daily_ingest × (1 / compression_ratio) × retention_days × safety_factor (1.25)` for each backend.
  - Derive Gateway VMSS sizing: benchmark a single B2ms at peak ingest; set VMSS autoscale min/max based on `measured_throughput_MB_s / instance_capacity_MB_s`.
  - Update `presentation.md` cost and Managed Disk tables with empirically derived figures.
  - Set Azure Monitor alerts (A-05) to alert at 80% disk utilisation — this is the primary early warning signal before capacity planning is formalised.
  - Evaluate Azure Reserved Instances (1-year or 3-year) for stable workload VMs once capacity is validated.

---

## F6. Summary Table — Config Source Column

- **Origin:** arch_eval.md P3 #22
- **Scope decision:** Deployment tooling and config distribution mechanisms are out of scope for the current high-level architectural design. How each collector type receives its configuration depends on tooling choices (Ansible, Azure DevOps Pipelines, Helm, etc.) that are resolved during production hardening, not during initial architecture design.
- **Context:** The `endpoints.md` Summary table currently lists deployment count, OS, signals, and ingest direction. A "Config Source" column would document how each deployment type receives its OTel Collector config — e.g., Ansible playbook via Azure DevOps Pipeline (Azure VM), Kubernetes ConfigMap via Helm (AKS DaemonSet), ACA container app revision env vars or Key Vault reference (ACA sidecar), embedded in function code (Azure Functions), and Ansible/cloud-init (DigitalOcean).
- **Why it matters (when revisited):** Operators frequently ask how to update a collector config in production. Without a documented config distribution path per deployment type, config changes are applied inconsistently or incorrectly across the fleet.
- **Future implementation path:**
  - Confirm the config distribution tool for each environment (Ansible via Azure DevOps, Helm, Terraform, Azure App Configuration, etc.).
  - Add a "Config Source" column to the Summary table in `endpoints.md` with an entry per deployment type.
  - Cross-reference each entry to the relevant deployment section where credentials, paths, and reload mechanisms are documented.

---

## F7. Failure Mode Documentation

- **Origin:** arch_eval.md P3 #24
- **Scope decision:** Failure modes and operator response runbooks are out of scope for the current high-level architectural design. Real operational experience and the monitoring tooling selected during production hardening (e.g., Azure Monitor Alerts, Action Groups, PagerDuty runbooks) should inform this documentation before it is written.
- **Context:** No documentation currently exists for failure scenarios: Gateway Azure LB backend pool draining to zero, VPN tunnel loss severing on-prem or DigitalOcean agents, Prometheus remote-write saturation causing drops, Loki unavailability silently discarding logs, or a VMSS scaling event leaving a temporary capacity gap.
- **Why it matters (when revisited):** During an incident, operators need to know immediately whether a gap in dashboards is a real outage or a telemetry pipeline failure. Without documented failure modes, triage time increases and false-negative confidence in the observability platform erodes.
- **Future implementation path:**
  - For each failure scenario, document: the expected OTel Collector behaviour (in-memory queue, drop-on-overflow, retry with backoff), how long data loss persists before the pipeline self-recovers, the alerting signal that fires (cross-reference `alerting.md`), and the operator remediation steps.
  - Key scenarios to cover: Gateway down (all agents queue locally, backpressure → drops after queue exhaustion), Azure LB backend pool empty (TCP connection refused → agent retries with exponential backoff), VPN Gateway tunnel drop (same as Gateway down for on-prem / DO agents), Prometheus remote-write saturation (Gateway export queue grows → A-03 alert fires; reduce scrape intervals or scale Prometheus), Loki unresponsive (log export errors → A-02 alert fires; check Loki disk — A-05).
  - Consider Azure Monitor Alert rules or Azure Automation Runbooks to automate detection of pipeline failure conditions and trigger automated remediation (e.g., Action Group → Azure Function to restart a crashed collector agent via Azure Automation, or auto-scale the Gateway VMSS on queue-depth breach via custom autoscale rules).
  - Publish as a `failure-modes.md` or a dedicated section in `endpoints.md` once production patterns are understood.
