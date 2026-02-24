# Implementation Phases (Azure)

Phased rollout plan for the centralized observability platform on Azure.

> **Resource assumption:** A single engineer owns all aspects of this project — stakeholder coordination, infrastructure provisioning, pipeline builds, agent deployments, instrumentation, and ongoing tuning. All phases are **sequential**. No parallel workstreams are assumed.
>
> **Deployment tooling:** Azure DevOps Pipelines with Terraform (infrastructure) and Ansible (VM-level configuration). Source code and config templates stored in Azure Git Repos.

---

## Phase 1 — Foundation & First Signal (Weeks 1–6)

Phase 1 establishes the entire foundation: distribution pipelines, core backend infrastructure, all Azure-hosted data sources, and the first dashboards and alerts. Every subsequent phase depends on this foundation being stable and trusted.

### Week 1: Prerequisites & Distribution Pipelines

Confirm or stand up the mechanisms needed to distribute the OTel Collector binary and configuration to every host type in the estate. Most items here are confirmations, not builds — with full access, the engineer resolves these in days, not weeks.

### Config Distribution for Host Agents

Assess what exists today and establish the minimum viable mechanism:

- **Azure VMs (Linux and Windows):** Use **Azure DevOps Pipelines triggering Ansible playbooks** for configuration distribution. Ansible connects via SSH (Linux) or WinRM (Windows) to Azure VMs. For hosts where Ansible connectivity is complex, consider the **Azure VM Custom Script Extension** as a lightweight alternative for initial binary deployment.
- **On-prem and DigitalOcean hosts:** These are outside Azure. **Azure Arc-enabled servers** can extend Azure management capabilities to non-Azure machines, enabling VM extensions and Azure Automation Runbooks. Alternatively, Ansible reach over the VPN/WireGuard path established in Phase 2 handles config distribution. If VPN is not yet in place, Ansible playbooks for remote hosts are written in Phase 2 Week 7 alongside the tunnel work.
- **Collector binary and config storage:** Create an Azure Blob Storage container (or Azure Container Registry for container images) to host `otelcol-contrib` binaries and versioned config files. Distribution scripts pull from this location. This decouples the distribution mechanism from the binary source and enables config-only updates without re-deploying the binary.

### CI/CD for Container and Kubernetes Deployments

- **AKS:** Confirm `kubectl` / Helm access to each cluster from the Azure DevOps agent pool or the engineer's workstation. The OTel DaemonSet will be deployed via Helm chart (`opentelemetry-helm-charts/opentelemetry-collector`); ensure the chart registry is reachable from AKS.
- **Azure Container Apps:** Confirm access to update container app configurations (via Azure CLI, Terraform `azurerm_container_app`, or the Azure Portal). The sidecar pattern requires a container app revision update per app; know the deployment mechanism before first data source deployment in Week 5.
- **Azure Functions:** Confirm access to update Function App configurations and deploy code packages via Azure DevOps Pipelines.

### TLS Certificates

- Provision or confirm TLS certificates in **Azure Key Vault** for `gateway.company.com` and `grafana.company.com`. These can be Key Vault-managed certificates (auto-renewal) or imported certificates.
- Azure Front Door provides free managed TLS certificates for custom domains — confirm the custom domain `grafana.company.com` can be validated via Azure DNS.
- OTel Gateway instances will retrieve TLS certificates from Key Vault via Managed Identity at startup.

### Access and Permissions

- Confirm Azure Managed Identities exist (or are ready to be created) for: Gateway VMSS VMs (Key Vault access, internal networking), backend VMs (Blob Storage access for Loki/Tempo), Remote Collection Scraper VM (Azure Monitor Reader, Key Vault access).
- Confirm Microsoft Entra ID tenant is provisioned and an appropriate directory group structure exists for Grafana role mapping (Viewer / Editor / Admin).
- Confirm PagerDuty and Slack webhook URLs are available for the Action Group setup in Phase 1 Week 6.
- Confirm Azure subscription quotas for the required VM SKUs (B2ms, B2s) in the target region.

*Exit criterion: all items above are confirmed in writing; no ambiguous dependencies remain before Week 2 POC work begins.*

---

### Week 2: Proof of Concept & Dev Validation

Before committing production sizing and fleet-wide rollout, validate the full pipeline end-to-end in a minimal, isolated environment. This phase de-risks the production deployment and gives stakeholders an early Grafana demo to align on expectations.

**Minimal dev environment**
- Spin up a single Azure VM running all components locally: `otelcol-contrib` (Gateway mode), Prometheus, Loki, and Tempo — no Azure LB, no VMSS, no HA. The goal is pipeline correctness, not production architecture.
- Use Docker Compose or a simple systemd setup; tear down after this phase.
- Confirm all three backends accept ingest from the Collector and are queryable in Grafana (using the production Azure Managed Grafana workspace pointed at dev endpoints, or a temporary local Grafana instance).

**One of each source type**
- Deploy one `otelcol-contrib` agent on a single non-critical Linux VM; confirm metrics and logs flow end-to-end to Prometheus and Loki.
- Run `postgresqlreceiver` or `mysqlreceiver` against a non-production Azure Database instance; confirm metrics land in Prometheus.
- Deploy the ACA sidecar pattern in a non-production container app; confirm container telemetry flows to the dev pipeline.

**TLS and auth validation**
- Validate the full TLS path: OTel agent → Azure LB (TCP pass-through) → Gateway (TLS termination using Key Vault cert). Confirm end-to-end encryption works.
- Walk through the Entra ID SSO login flow on Azure Managed Grafana to confirm the authentication, role assignment (Grafana Viewer/Editor/Admin via Azure RBAC), and dashboard access work as expected.

**Stakeholder demo**
- Present at least one working Grafana dashboard with live data to a stakeholder before production rollout begins. Confirm alignment on what the finished dashboards should show.

*Exit criterion: data flows from agent → Gateway → all three backends → Grafana with no gaps; TLS and Entra ID auth validated; stakeholder sign-off on dashboard direction received.*

---

### Week 3–4: Backend Infrastructure

Stand up the production storage backends and the Gateway tier. This is the most infrastructure-heavy block in the project — Azure LB + VMSS + TLS, three backend VMs, and Managed Grafana + Front Door + WAF + Entra ID all need to come online. Two weeks accounts for inevitable networking and NSG debugging.

**OTel Gateway (Azure LB + VMSS)**
- Provision the Azure Load Balancer Standard with a static public IP; configure TCP load-balancing rules on port 443 mapped to OTLP gRPC (4317) and HTTP (4318) on Gateway VMSS instances.
- Configure the health probe to target `HTTP :13133/health` (OTel Collector healthcheck extension).
- Launch the OTel Gateway VMSS with `otelcol-contrib`; configure a minimal pass-through pipeline first (receive → batch → export) to validate end-to-end connectivity before enabling enrichment and sampling rules.
- Configure TLS termination on Gateway instances using certificates fetched from Azure Key Vault via Managed Identity.
- Configure VMSS autoscale rules based on CPU utilisation and custom metrics (OTel Collector queue depth published via AMA or Prometheus).

**Prometheus**
- Deploy Prometheus on an Azure VM with a Premium SSD v2 Managed Disk sized for the 90-day TSDB target (~150 GB).
- Confirm remote-write endpoint is reachable from the Gateway VMSS subnet (NSG rules).

**Grafana Loki**
- Deploy Loki in simple-scalable or monolithic mode; configure Premium SSD v2 for WAL and index (~20 GB), Azure Blob Storage container for chunk storage (~36 GB target at 90 days).
- Configure Loki to use Managed Identity for Blob Storage access (no storage account keys).
- Confirm Loki OTLP/HTTP ingest endpoint is reachable from the Gateway.

**Grafana Tempo**
- Deploy Tempo; configure Premium SSD v2 for WAL (~10 GB), Azure Blob Storage container for trace blocks (~18 GB target at 90 days).
- Configure Tempo to use Managed Identity for Blob Storage access.
- Confirm Tempo OTLP ingest endpoint is reachable from the Gateway.

**Azure Managed Grafana**
- Provision the Managed Grafana workspace (Standard tier); Entra ID SSO is automatic.
- Configure Azure Front Door Standard with a WAF policy for `grafana.company.com:443`; enable rate-limiting and geo/IP rules.
- Map Entra ID groups to Grafana roles via Azure RBAC role assignments (`Grafana Viewer`, `Grafana Editor`, `Grafana Admin`).
- Configure managed private endpoints from Grafana to backend VMs (Prometheus :9090, Loki :3100, Tempo :3200).
- Add Prometheus, Loki, and Tempo as datasources in Grafana; validate all three with test queries.

*Exit criterion: `otelcol-contrib` test agent → Azure LB → Gateway → backends → Grafana query returns data.*

---

### Week 5: First Data Sources (Azure-Hosted, Zero Code Changes)

Deploy OTel Collector agents to all Azure-hosted infrastructure using the distribution pipeline built in Week 1. The agent config is a near-identical template per host — this is repetitive distribution work, not engineering. These sources require no application changes.

**Azure Linux VMs (Intranet, B2B, ETL pipelines, scheduled jobs)**
- Install `otelcol-contrib` as a systemd service on each Linux VM via Ansible (triggered by Azure DevOps Pipeline).
- Enable `hostmetricsreceiver` (CPU, memory, disk, network) and `filelogreceiver` for application log paths.
- Configure each agent to export TLS OTLP to `gateway.company.com:443`.
- Apply resource detection processor (`resourcedetectionprocessor` with `azure` detector) to attach VM name, resource group, region, zone, and service identity attributes.

**Azure Database for PostgreSQL and MySQL**
- Enable `postgresqlreceiver` and `mysqlreceiver` in the Remote Collection Scraper VM (or directly on host agents if DB hosts have agents).
- Enable `azuremonitorreceiver` for Azure Database metrics (storage, connections, latency) from Azure Monitor.

**AKS**
- Deploy `otelcol-contrib` as a DaemonSet via Helm chart; configure `kubeletstatsreceiver` for pod/node metrics and `filelogreceiver` for container logs via the node's `/var/log/pods` path.
- Apply OTel auto-instrumentation where supported via the OTel Operator `Instrumentation` CRD.

**Azure Container Apps**
- Add `otelcol-contrib` as a sidecar container in each container app revision.
- Configure the sidecar to receive OTLP from the application container on localhost and forward to the Gateway Azure LB.

*Exit criterion: host metrics, database metrics, and container logs are visible in Grafana for all Azure-hosted systems.*

---

### Week 6: First Dashboards and Alerts

With data flowing, stand up the minimum viable dashboards and safety-net alerting.

**Dashboards**
- Host health: CPU, memory, disk utilisation, and system load — one dashboard per environment tag.
- Database health: connections, query latency, replication lag, storage headroom — for Azure Database for PostgreSQL and MySQL.
- AKS / ACA container health: pod/task status, restarts, CPU and memory per workload.
- OTel Pipeline health: Gateway queue depth, export error rate, dropped spans/metrics/logs, Azure LB health.

**Azure-Native Safety-Net Alerts (Path 2)**
- Azure Monitor Alerts: Azure DB disk saturation, ACA sidecar restart detection (A-01), Azure LB unhealthy backends (A-04), VM disk utilisation on backend nodes (A-05), VMSS instance count below floor (A-06).
- Route all alerts through Action Groups → PagerDuty / Slack webhooks.

**Grafana Managed Alerts (Path 1)**
- OTel Gateway export errors (A-02), queue saturation (A-03).
- Prometheus ingestion health (A-07), Loki ingestion health (A-08), Tempo ingestion health (A-09).
- Route through Grafana contact points → webhook → Action Groups.

*Exit criterion: all Phase 1 alerts are firing correctly in test conditions; on-call rotation receives notifications via both paths.*

---

## Phase 2 — Expanded Coverage (Weeks 7–9)

The platform is proven. Expand to remaining infrastructure, connect external data sources, and begin trace rollout.

### Week 7: Remote & On-Prem Connectivity

Agent deployment reuses the same Ansible pattern and config templates from Phase 1. The main effort is establishing the network paths — VPN/WireGuard tunnel setup and routing validation.

**DigitalOcean Hosts (scheduled jobs)**
- Establish site-to-site WireGuard tunnel between DigitalOcean and the Azure VNet.
- Deploy `otelcol-contrib` on each DO host; configure `hostmetricsreceiver` and `filelogreceiver`; export TLS OTLP through the tunnel to `gateway.company.com:443`.
- Optionally register DO hosts as Azure Arc-enabled servers for centralised management.

**On-Premises Hosts (databases, Windows hosts)**
- Provision the Azure VPN Gateway (VpnGw1 SKU); establish S2S IPsec VPN tunnel to on-prem.
- Deploy `otelcol-contrib` on Linux on-prem database hosts using the same systemd pattern as Azure Linux VMs.
- Deploy `otelcol-contrib` as a Windows service on Windows hosts; configure `hostmetricsreceiver` with Windows-specific performance counter targets and `windowseventlogreceiver` for Windows Event Log.
- Enable `postgresqlreceiver` / `mysqlreceiver` for on-prem database instances.

### Week 8: SaaS and API Sources

Each source follows the same pattern: HTTP polling or webhook receiver → OTLP → Gateway. The integration work per source is 1–2 days including its dashboard.

**Shopify (Ecommerce)**
- Configure a webhook receiver on the Remote Collection Scraper VM (`httplogreceiver`) to accept Shopify order and fulfilment events.
- Alternatively, poll the Shopify Admin API on a schedule and push metrics into the pipeline via an OTLP exporter in a small Python script.
- Produce a Grafana dashboard for order volume, fulfilment rate, and checkout error rate.

**POS (Retail, REST API)**
- Configure API polling on the Remote Collection Scraper; extract transaction counts and error rates.
- Add a POS health dashboard alongside the ecommerce dashboard.

**Snowflake (Data Warehouse)**
- Poll Snowflake usage and audit query APIs via the `snowflakereceiver`; extract credit consumption, query duration, and warehouse utilisation metrics.
- Forward via OTLP to the Gateway; build a Snowflake cost and performance dashboard.

### Week 9: Azure Functions and Initial Trace Rollout

Azure Functions OTel SDK integration requires code changes (adding OTel SDK dependencies), unlike the AWS ADOT Lambda Layer which is config-only. Budget an extra day for this step.

**Azure Functions**
- Add OTel SDK packages (`opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-grpc`) to each function app's dependencies.
- Initialize the OTel SDK at function app startup; configure OTLP export to `gateway.company.com:443`.
- Validate that traces, metrics, and logs flow from Azure Functions through the Gateway to Tempo and Loki.
- Deploy via Azure DevOps Pipelines; pin OTel SDK versions in the function project files.

**First Trace Rollout (one service)**
- Choose the highest-value, lowest-risk service as the first trace target — recommended: B2B website or Intranet (JS/PHP, running on an already-instrumented Azure VM).
- Use OTel auto-instrumentation for PHP (`opentelemetry-auto-*` packages) and the OTel JS browser/Node SDK to minimise code changes.
- Configure the local OTel Collector agent as the trace receiver on localhost; the agent forwards to the Gateway via TLS OTLP.
- Validate end-to-end trace visibility in Tempo; confirm trace-to-log correlation via `trace_id` attribute in Loki.

*Exit criterion: all infrastructure sources are covered; at least one service has full trace coverage in Tempo.*

---

## Phase 3 — Maturity & Tuning (Weeks 10–13)

Operationalise the platform and transfer ownership to teams for self-service use.

### Week 10–11: Complete Trace Rollout

Auto-instrumentation covers most services with minimal code changes. Manual span creation is reserved for business-critical boundaries only. Two weeks accounts for the volume of services and per-service validation.

- **Python ETL pipelines and scheduled jobs:** Add `opentelemetry-sdk` and `opentelemetry-instrumentation` to each Python service. Use `opentelemetry-instrumentation-auto` where available; manual span creation only for business-critical boundaries.
- **Azure Container Apps services:** Instrument application containers to emit OTLP traces to the sidecar collector on localhost.
- **AKS workloads:** Apply the OTel Operator and `Instrumentation` CRD for language-level auto-instrumentation where supported; manual SDK for remaining services.
- For each service with traces enabled, validate trace-to-log and trace-to-metric correlation in Grafana Explore.

### Week 12: Alerting Maturity

Technical implementation — recording rules, alert rules, PagerDuty routing — is 3–4 days. The variable is stakeholder alignment on SLO targets.

- **SLO-based alerting:** Define SLOs for key services (e.g. B2B checkout availability ≥ 99.5%, API p99 latency < 500 ms). Implement Grafana recording rules and alerting rules that evaluate against these targets.
- **Runbook links:** Attach a `runbook_url` annotation to every alert rule, pointing to an internal runbook in the company wiki.
- **Escalation tiers in PagerDuty:** Configure severity-based routing — Critical alerts page on-call immediately; Warning alerts enter a low-urgency queue that notifies Slack only during business hours.
- **Alert deduplication:** Review firing alert volume; suppress noisy derived alerts that duplicate upstream root-cause alerts.

### Week 13: RBAC Rollout + Cost Review and Retention Tuning

Two smaller workstreams combined into one week. RBAC is 2–3 days of configuration (Entra ID role mapping was already validated in the POC). Cost review uses ~10 weeks of observed data to extrapolate 90-day storage and adjust sizing.

**RBAC (first half of week)**

- Map Entra ID groups to Azure RBAC role assignments on the Grafana resource: `Grafana Viewer` (all staff), `Grafana Editor` (DevOps / SRE), `Grafana Admin` (platform owner only).
- Create team-scoped dashboard folders: Ops team sees infrastructure dashboards; Dev teams see application/service dashboards; Finance sees cost dashboards.
- Restrict direct Explore access to Prometheus, Loki, and Tempo datasources to `Editor+` roles.
- Validate that a Viewer-role test account cannot access restricted datasources or folders.

**Cost Review and Retention Tuning (second half of week)**

- Review actual storage consumption against Phase 1 estimates (150 GB Prometheus, 56 GB Loki, 28 GB Tempo at 90 days).
- Adjust Prometheus TSDB `--storage.tsdb.retention.time` and block compaction settings if consumption is trending above estimate.
- Review Loki chunk compaction schedule; move chunks older than 30 days to Blob Storage Cool tier via lifecycle management rules.
- Review Tempo block TTL; move cold blocks to Blob Storage Cool tier if read frequency is low.
- Review OTel Gateway VMSS sizing: if p95 CPU is consistently below 40%, reduce the minimum instance count.
- Produce a one-page cost summary: actual monthly spend on VMs (Gateway + backends), Managed Disks, Blob Storage, Managed Grafana, Azure LB, Front Door, and data transfer versus initial estimates.
- Evaluate reserved instance commitment opportunity based on observed stable workload.

*Exit criterion: all data sources covered, full trace rollout complete, SLO alerting live, RBAC enforced, 90-day retention validated against budget.*

---

## Summary Timeline

| Phase | Weeks | Focus |
|-------|-------|-------|
| 1 | 1 | Prerequisites: Azure DevOps Pipelines + Ansible setup, AKS/ACA/Functions access, Blob Storage, Key Vault certs, Managed Identities, notification tokens |
| 1 | 2 | Proof of concept: end-to-end pipeline validation in dev, TLS/Entra ID walkthrough, stakeholder demo |
| 1 | 3–4 | Core backend infrastructure (Gateway Azure LB + VMSS, Prometheus, Loki, Tempo, Managed Grafana + Front Door + WAF + Entra ID) |
| 1 | 5 | First data sources: all Azure-hosted VMs, Azure Database, AKS, ACA |
| 1 | 6 | First dashboards and safety-net alerting (both paths live) |
| 2 | 7 | Remote connectivity: DigitalOcean WireGuard + on-prem VPN Gateway + host agents |
| 2 | 8 | SaaS and API sources: Shopify, POS, Snowflake |
| 2 | 9 | Azure Functions OTel SDK + first trace rollout (one service) |
| 3 | 10–11 | Complete trace rollout across all services |
| 3 | 12 | Alerting maturity: SLOs, runbooks, escalation tiers |
| 3 | 13 | RBAC rollout + cost review, retention tuning, handover documentation |
