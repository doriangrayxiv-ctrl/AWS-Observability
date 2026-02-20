# GitHub Copilot Instructions

## Purpose
- Provide repository-specific guidance so all Copilot agents produce artifacts aligned with Brighton Collectibles' chosen observability stack and project goals.

## Scope
- Covers architecture, configuration, scripting, documentation, and rollout planning for the centralized observability platform.
- Intended audience: DevOps engineers and Copilot agents generating code, IaC, diagrams, and written deliverables.

## Project Objective
- Design and implement a **cost-effective, centralized observability platform** that unifies **metrics, logs, and traces** across all Brighton Collectibles systems.
- The platform must enable **self-service dashboards and alerting** and provide application- and infrastructure-level health awareness.
- Reference `original.md` for the full exercise brief, constraints, and deliverables.

## Chosen Technology Stack

All generated artifacts, configs, and recommendations **must** target this stack:

| Pillar | Technology | Notes |
|--------|-----------|-------|
| **Collection** | OpenTelemetry Collector Contrib (`otelcol-contrib`) | Deployed on both **Linux and Windows** hosts as agents/sidecars |
| **Metrics** | **Prometheus** | Remote-write target from OTel Collector; long-term store for all metrics |
| **Logs** | **Grafana Loki** | Receives logs from OTel Collector via Loki exporter or OTLP |
| **Traces** | **Grafana Tempo** | Receives traces via OTLP from OTel Collector |
| **Visualization & Alerting** | **Grafana (on-prem)** | Single pane of glass for dashboards, alerting, and exploration |

### Collection Architecture
- **OpenTelemetry Collector Contrib** is the **sole collection agent**. Do not suggest Fluentd, Filebeat, Telegraf, Datadog, or other proprietary/alternative collectors.
- OTel Collectors run on every host (Linux and Windows) and as sidecars in Kubernetes (EKS) and ECS Fargate.
- Use OTel Collector **receivers** to ingest data from each source system (see below).
- Use OTel Collector **processors** for parsing, enrichment, filtering, and batching.
- Use OTel Collector **exporters** to route telemetry to the correct backend (Prometheus, Loki, Tempo).

### Data Sources & Integration Points
When generating configs or architecture references, account for these systems:

| System | Hosting | Integration Approach |
|--------|---------|---------------------|
| Ecommerce (Shopify) | Shopify SaaS | Webhook/API polling; limited to what Shopify exposes |
| Retail POS (REST API) | External | API polling or webhook receiver in OTel Collector |
| Intranet (JS/PHP) | AWS | OTel Collector agent on host; language SDKs for traces |
| B2B Website (JS/PHP) | AWS | OTel Collector agent on host; language SDKs for traces |
| Data/ETL Pipelines (Python) | AWS | OTel Python SDK for traces/metrics; Collector for logs |
| Scheduled Jobs (Python) | AWS + DigitalOcean | OTel Collector agent on each host |
| Data Warehouse | Snowflake SaaS | Query Snowflake usage/audit APIs; ingest via Collector |
| Databases (Postgres/MySQL) | On-prem + AWS RDS | OTel Collector `postgresqlreceiver`, `mysqlreceiver` |
| AWS EKS | AWS | DaemonSet OTel Collector + sidecar pattern |
| AWS ECS Fargate | AWS | Sidecar OTel Collector container |
| AWS Lambda | AWS | OTel Lambda layer |
| AWS RDS | AWS | CloudWatch metrics via `awscloudwatchreceiver` |

### Connectivity
- **On-prem → Central stack**: VPN or direct-connect tunnel to the Grafana/Prometheus/Loki/Tempo endpoints.
- **DigitalOcean → Central stack**: Site-to-site VPN or WireGuard tunnel; OTel Collectors on DO hosts export over the tunnel.
- All telemetry transport should use **TLS-encrypted OTLP (gRPC or HTTP)** where possible.

## Style & Constraints
- Keep outputs concise, actionable, and targeted at a DevOps audience.
- Prioritize **minimal application changes**: prefer agents, sidecars, auto-instrumentation, and platform-native telemetry over manual SDK instrumentation.
- Always recommend **open-source, cost-effective** solutions consistent with the stack above.
- Default log/telemetry retention: **90 days**.
- The design must be **reliable enough to depend on during incidents**.

## Deliverables Checklist
1. **Architecture Diagram** — data sources → OTel Collectors → processors → Prometheus / Loki / Tempo → Grafana. Include on-prem and DigitalOcean connectivity.
2. **Written Rationale (2–3 paragraphs)** — justify the stack choice, assumptions, minimal-change approach, and how logs + metrics + traces are handled.
3. **Rollout Plan / Project Phasing** — phased approach; first 2–4 weeks deliver quick value, later phases expand coverage.

## Prompt Examples (aligned to stack)
- "Generate an OTel Collector Contrib config (`otelcol-contrib.yaml`) that scrapes Prometheus metrics from an EKS cluster and exports to a remote Prometheus instance."
- "Write a `docker-compose.yml` for local testing with otelcol-contrib, Prometheus, Loki, Tempo, and Grafana."
- "Create a phased rollout plan (weeks 1–4, 5–8, 9–12) for deploying OTel Collectors across AWS, on-prem, and DigitalOcean."
- "Draft a 2–3 paragraph rationale for choosing Grafana + OTel over Datadog/Splunk for a mixed AWS/on-prem/DO environment."
- "Generate a Grafana dashboard JSON for visualizing OTel Collector health (queue length, export errors, spans received)."

## Verification & Follow-ups
- After generating content, validate: correct OTel Collector component names (receivers, processors, exporters), security posture, credential handling, and cost assumptions.
- Ensure all configs reference `otelcol-contrib` (not the core distribution) since contrib includes the required receivers.
- Ask for a test plan for any generated scripts or IaC.

## Contacts / Notes
- Reference `original.md` for the full exercise constraints and deliverables.

