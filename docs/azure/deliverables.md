# Deliverables (Azure)

High-level observability architecture document — Azure translation of the original AWS design.

---

## 1. Architecture Diagram

> High-level architecture only. Fine-grained details (storage sizing, OTel Collector config, IaC) are out of scope for this diagram.

- **Data Sources** — Shopify, Retail POS, Intranet, B2B website, ETL pipelines, scheduled jobs, Snowflake, Postgres/MySQL, AKS, Azure Container Apps, Azure Functions, Azure Database for PostgreSQL/MySQL
- **Collection** — OTel Collector Contrib agents on Linux and Windows hosts; DaemonSet on AKS; sidecar containers on Azure Container Apps; OTel SDK in Azure Functions
- **Transport** — TLS-encrypted OTLP from agents → Azure Load Balancer Standard → OTel Gateway VMSS; on-prem via Azure VPN Gateway; DigitalOcean via WireGuard tunnel
- **Gateway Processing** — OTel Collector gateway tier on Virtual Machine Scale Set; tail-sampling, PII scrubbing, attribute normalization, metric/span filtering, batching before export
- **Storage Backends** — Prometheus (metrics, TSDB on Managed Disks); Grafana Loki (logs, WAL/index on Managed Disks + chunks in Azure Blob Storage); Grafana Tempo (traces, WAL on Managed Disks + blocks in Azure Blob Storage)
- **Visualization** — Azure Managed Grafana at `grafana.company.com:443`; unified dashboards across metrics, logs, and traces; self-service for Viewer/Editor roles
- **Access Controls** — Azure Front Door with WAF (rate-limiting, geo/IP filtering, bot protection) → Azure Managed Grafana with Entra ID SSO (built-in authentication) → Grafana RBAC (Viewer/Editor/Admin roles via Azure RBAC assignments, folder-level permissions, team-based assignments, data source access restrictions)

---

## 2. Processing Strategy

Not all processing is deferred to the Gateway. Edge collectors perform a limited set of lightweight operations where it makes sense to do so before data leaves the host:

- **Format normalisation** — some sources emit data in non-OTLP formats (e.g. Prometheus exposition, Windows Event Log XML, syslog). The edge collector transforms these into OTLP before forwarding, so the Gateway and all downstream components deal with a single, consistent wire format.
- **Coarse filtering** — verbose sources (e.g. debug-level application logs, high-cardinality trace spans from noisy background threads) can be filtered at the edge to avoid saturating the transport path. Only a representative subset is forwarded rather than the full volume.
- **Deduplication** — where an agent collects multiple repeated events in a given interval, duplicates are tallied+merged before forwarding.

Edge processing is intentionally minimal — the goal is to keep agent configs simple and host-resource overhead low. No PII handling, tail-sampling, or complex routing logic belongs at the edge.

The **OTel Gateway cluster** (VMSS behind the Azure LB) is the primary processing tier. All telemetry from every source converges here before reaching a storage backend. The Gateway is responsible for:

- **Tail-based trace sampling** — decisions made after the full trace is assembled, not per-span at the edge
- **PII scrubbing** — attribute processors redact or hash sensitive fields before data is written to any store
- **Attribute normalisation** — service names, environment labels, and resource attributes are standardised across all sources
- **Metric and span filtering** — fine-grained drop rules applied centrally rather than duplicated across every agent
- **Routing** — telemetry is fanned out to the correct backend (Prometheus, Loki, or Tempo) based on signal type
- **Batching** — output is batched and compressed for efficient backend ingest

---

## 3. Alerting Strategy

The alerting design follows a **two-path model** that separates concerns by signal fidelity and delivery dependency:

- **Path 1 — Azure-Native (Event Grid + Azure Monitor Alerts):** Detects infrastructure and container lifecycle events directly from the Azure control plane. This path operates independently of the observability stack and fires even when the OTel pipeline is degraded or completely down — making it the safety net for pipeline-level failures (ACA sidecar crash, Azure LB unhealthy backends, VMSS instance floor, disk saturation).
- **Path 2 — Grafana Managed Alerting (Azure Managed Grafana):** Evaluates metric and log signal thresholds using Prometheus, Loki, and Azure Monitor datasources. This path provides richer, application-aware alerting but depends on the pipeline being healthy — used for OTel Gateway export errors, queue saturation, and backend ingestion health.

Both paths can route notifications through **Azure Monitor Action Groups → PagerDuty / Slack (webhooks) / Email**, ensuring a single operator notification workflow regardless of which path fires.

---

## 4. Written Rationale (2–3 paragraphs)

### Rationale

A few sizing assumptions anchored this design. At peak I expected roughly **1,000+ concurrent OTel Collector connections** hitting the Gateway, generating around **8 GB of telemetry per day**. On the consumption side I assumed a small team — approximately **10 concurrent Grafana users** — with access segmented by role (Viewer / Editor / Admin) through Microsoft Entra ID. Working from those numbers, 90-day retention lands at roughly 150 GB for Prometheus, 56 GB for Loki (index on Managed Disks, chunks in Blob Storage), and 28 GB for Tempo (WAL on Managed Disks, blocks in Blob Storage) — about 234 GB in total. Some assumptions were made for current architecture — for example, whether scheduled scripts emit structured logs or need an SDK import — I called out the ambiguity and designed for the more conservative case rather than assuming the best one.

The stack retains the same open-source storage and collection components — **OpenTelemetry Collector Contrib**, **Prometheus**, **Grafana Loki**, and **Grafana Tempo** — running on Azure infrastructure instead of AWS. The central choice of OTel Collector Contrib as the sole collection agent carries the same advantages: a single binary covers every data source in the estate, one config surface, one upgrade path, and lower per-host overhead. The Azure-specific changes centre on infrastructure: **Azure Load Balancer Standard** (L4 TCP) replaces the AWS NLB for push traffic at dramatically lower cost (~$20/mo vs $50–$120/mo), though TLS termination moves from the load balancer to the Gateway instances — providing true end-to-end encryption. **Azure Managed Grafana** replaces AWS Managed Grafana; it integrates Entra ID SSO natively without requiring an ALB+OIDC layer, simplifying the dashboard access path. **Azure Front Door Standard + WAF** replaces the ALB+WAF combination for dashboard user protection. The Azure VPN Gateway (VpnGw1 SKU) is more expensive than AWS Site-to-Site VPN (~$140/mo vs ~$72/mo), which is the primary cost delta; however, the overall architecture remains cost-competitive at ~$410–$540/mo with reserved instances. Azure DevOps Pipelines with Terraform and Ansible manage all IaC and configuration, with source in Azure Git Repos.

The three signal types — metrics, logs, and traces — each take a slightly different path to full coverage, which is intentional. Metrics and logs flow from day one with minimal or no application changes: OTel Collector agents read what the infrastructure already produces and forward it through the Gateway. Traces are a different story — they require at least a small instrumentation step per service, so they roll out incrementally. The most notable platform difference from AWS is Azure Functions: unlike Lambda's managed ADOT Layer, Azure Functions require direct OTel SDK integration in the function code, making serverless instrumentation slightly more hands-on but equally functional. The platform delivers immediate, broad visibility on day one and deepens as trace instrumentation is added service by service.

---

## 5. Rollout Plan / Project Phasing

> **Resource assumption:** A single engineer owns all aspects — coordination, implementation, pipeline builds, and stakeholder communication.

It's worth being honest about the timeline: 4 weeks to value is optimistic for one engineer carrying this end-to-end. It assumes no interruptions, no competing priorities, no procurement delays (VPN provisioning, Entra ID onboarding, VPN Gateway lead times), and no surprises in the existing infrastructure. In practice, a more realistic delivery window for a solo implementer is **5 weeks** for value ramp up and closer to **16–20 weeks** for completion when accounting for coordination overhead across multiple teams, waiting on approvals, and the debugging that comes with connecting systems that have never connected in this manner. The phasing below reflects the most efficient sequential order — not a guarantee of pace.

The first two weeks are entirely setup — confirming access, standing up distribution pipelines, and running a proof-of-concept to validate the pipeline before committing to production infrastructure. Weeks 3 and 4 are backend infrastructure: the Gateway, storage backends, and Grafana. None of this produces dashboards anyone can use yet. **The first real, visible value arrives at Week 5**, when agents are deployed to Azure hosts and real metrics and logs start flowing into Grafana for the first time.


| Phase | Week | Focus |
|-------|-----:|-------|
| 1 — Foundation | 1 | Prerequisites: Azure DevOps Pipelines + Ansible setup, AKS/ACA/Functions access, Blob Storage artifact container, Key Vault certs, Managed Identities, notification tokens |
| 1 — Foundation | 2 | POC / dev validation: end-to-end pipeline on a single Azure VM, TLS + Entra ID auth walkthrough, stakeholder demo and sign-off |
| 1 — Foundation | 3–4 | Production backend infra: Gateway Azure LB + VMSS, Prometheus, Loki, Tempo, Azure Managed Grafana (Front Door + WAF + Entra ID) |
| 1 — Foundation | 5 | First Azure data sources: Linux VMs, Azure Database receivers, AKS DaemonSet, ACA sidecars — zero application code changes |
| 1 — Foundation | 6 | First dashboards (host, DB, container, pipeline health) and safety-net alerting (both Azure Monitor and Grafana paths live) |
| 2 — Expanded coverage | 7 | Remote connectivity: DigitalOcean WireGuard tunnel + on-prem VPN Gateway; agent deployment to remote hosts |
| 2 — Expanded coverage | 8 | SaaS and API sources: Shopify webhook/polling, POS API polling, Snowflake audit API |
| 2 — Expanded coverage | 9 | Azure Functions OTel SDK + first trace rollout on one high-value service (auto-instrumentation) |
| 3 — Maturity & tuning | 10–11 | Complete trace rollout across all services (auto-instrument where possible; manual SDK for business-critical boundaries) |
| 3 — Maturity & tuning | 12 | Alerting maturity: SLO-based alerts, runbook links, PagerDuty escalation tiers, alert deduplication |
| 3 — Maturity & tuning | 13 | Grafana RBAC rollout + cost/retention tuning + handover documentation |

---

## 6. Cost Analysis

### Inputs

| Input | Value |
|---|---|
| Total daily ingest | 8 GB/day |
| Retention | 90 days |
| Monitored endpoints | 1,000+ |
| Grafana editors | 10 users |
| Region | East US 2 |
| Instance sizing | Minimum viable |

### Signal Distribution

| Signal | Split | Daily Volume | 90-Day Total |
|---|---|---|---|
| **Logs** | 70% | 5.6 GB/day | 504 GB |
| **Metrics** | 20% | 1.6 GB/day | 144 GB |
| **Traces** | 10% | 0.8 GB/day | 72 GB |
| **Total** | 100% | 8 GB/day | 720 GB |

### Monthly Cost (East US 2)

| Component | Resource | Unit Cost | Monthly Cost |
|---|---|---|---|
| **OTel Gateway VMSS** | 2× B2ms (2 vCPU, 8 GB) Pay-as-you-go | ~$60/ea | $120 |
| **Gateway Azure LB Standard** | 1× LB + rules + data processing | ~$18 base + ~$1 processing | $20 |
| **Prometheus VM** | 1× B2ms (2 vCPU, 8 GB) | $60/mo | $60 |
| **Prometheus Managed Disk** | 150 GB Premium SSD v2 | ~$0.095/GiB/mo | $14 |
| **Loki VM** | 1× B2ms (2 vCPU, 8 GB) | $60/mo | $60 |
| **Loki Blob Storage** | 504 GB Hot tier | $0.018/GB/mo | $9 |
| **Loki Blob Requests** | PUT/GET ops | Estimated | $5 |
| **Tempo VM** | 1× B2s (2 vCPU, 4 GB) | $30/mo | $30 |
| **Tempo Blob Storage** | 72 GB Hot tier | $0.018/GB/mo | $1 |
| **Tempo Blob Requests** | PUT/GET ops | Estimated | $2 |
| **Azure Managed Grafana** | Standard tier (workspace, no per-editor charge) | ~$73/mo | $73 |
| **Azure Front Door Standard** | 1× profile + WAF policy | ~$35 + $5 WAF | $40 |
| **Azure VPN Gateway** | 1× VpnGw1 + S2S connections | ~$140/mo | $140 |
| **Data Transfer** | ~240 GB in (free) + minimal egress | Variable | $10 |
| **Azure Key Vault** | Certs + secrets + key operations | Minimal | $5 |
| **Azure DNS** | 1 zone | $0.50/zone/mo | $1 |
| **Total** | | | **~$590/mo** |

> Committing to 1-year Reserved Instances on Prometheus, Loki, Tempo, and Gateway VMs saves approximately 35% on those compute lines (~$95/mo). Blob Storage stabilises at 90-day rolling retention once lifecycle management rules are applied.
>
> **vs AWS estimate:** The Azure estimate (~$590 pay-as-you-go, ~$410–$540 reserved) is moderately higher than the AWS equivalent (~$432 pay-as-you-go, ~$255–$360 reserved). The primary cost driver is the Azure VPN Gateway ($140/mo vs AWS VPN $72/mo). Azure LB Standard is cheaper than AWS NLB ($20 vs $50–$120), and Azure Managed Grafana is cheaper than AWS AMG at 10 editors ($73 vs $90). Overall the platforms are cost-competitive within the same order of magnitude.
