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
| **Visualization & Alerting** | **AWS Managed Grafana** | Single pane of glass for dashboards, alerting, and exploration; accessed via `grafana.company.com:443` |

### Collection Architecture
- **OpenTelemetry Collector Contrib** is the **sole collection agent**. Do not suggest Fluentd, Filebeat, Telegraf, Datadog, or other proprietary/alternative collectors.
- OTel Collectors run on every host (Linux and Windows) and as sidecars in Kubernetes (EKS) and ECS Fargate.
- Use OTel Collector **receivers** to ingest data from each source system (see below).
- Use OTel Collector **processors** for parsing, enrichment, filtering, and batching at the agent level (lightweight: queuing, resource detection, basic attribute enrichment only).
- Use OTel Collector **exporters** to forward all telemetry via TLS OTLP to the OTel Gateway tier.

#### OTel Gateway Tier (centralized processing)
- All agent/sidecar telemetry is forwarded to an **AWS Network Load Balancer (NLB)** at **`nlb.company.com:443`** that fronts a pool of **OTel Gateway Collector servers running on an AWS Auto Scaling Group (ASG)**.
- The NLB provides TCP/TLS pass-through on port 443 (mapped to OTLP gRPC 4317 / HTTP 4318 on Gateway instances) and distributes load across Gateway instances.
- **OTel Gateway servers are responsible for all critical processing**: tail-sampling, span/metric filtering, PII scrubbing, attribute normalization, routing decisions, and batching before export.
- Gateway Collectors export to the appropriate backend:
  - **Prometheus remote-write** for metrics
  - **Loki exporter or OTLP** for logs
  - **OTLP to Grafana Tempo** for traces
- ASG scaling policies should be based on OTel Collector queue depth and CPU; ensure Gateway capacity is sized for peak ingest before incident conditions degrade it.
- Do not perform heavy processing on edge agents — keep agent configs minimal to reduce host overhead.

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
- **On-prem → Gateway NLB**: VPN or AWS Direct Connect tunnel; OTel agents export TLS OTLP to `nlb.company.com:443`.
- **DigitalOcean → Gateway NLB**: Site-to-site VPN or WireGuard tunnel; OTel Collectors on DO hosts export TLS OTLP over the tunnel to `nlb.company.com:443`.
- **All telemetry transport** must use **TLS-encrypted OTLP (gRPC or HTTP)**; no unencrypted telemetry paths.
- **Dashboard access**: Users reach AWS Managed Grafana at `grafana.company.com:443` via:
  1. **AWS WAF** — inspects and filters inbound HTTPS requests.
  2. **AWS Application Load Balancer (ALB)** — terminates TLS and enforces **OIDC authentication** before proxying to Managed Grafana.
  3. **AWS Managed Grafana** — receives only authenticated requests from the ALB.
- The ALB listener rule should require a valid OIDC session; unauthenticated requests receive a 401/redirect. Configure the WAF with rate-limiting and geo/IP rules appropriate for the dashboard user population.

## Style & Constraints
- Keep outputs concise, actionable, and targeted at a DevOps audience.
- Prioritize **minimal application changes**: prefer agents, sidecars, auto-instrumentation, and platform-native telemetry over manual SDK instrumentation.
- Always recommend **open-source, cost-effective** solutions consistent with the stack above.
- Default log/telemetry retention: **90 days**.
- The design must be **reliable enough to depend on during incidents**.

## Deliverables Checklist
1. **Architecture Diagram** — data sources → OTel Agent Collectors → NLB → OTel Gateway ASG (critical processing) → Prometheus / Loki / Tempo → AWS Managed Grafana. Include WAF → ALB (OIDC) → Managed Grafana user access path, and on-prem / DigitalOcean VPN connectivity to the Gateway NLB.
2. **Written Rationale (2–3 paragraphs)** — justify the stack choice, assumptions, minimal-change approach, and how logs + metrics + traces are handled.
3. **Rollout Plan / Project Phasing** — phased approach; first 2–4 weeks deliver quick value, later phases expand coverage.

## Prompt Examples (aligned to stack)
- "Generate an OTel Collector Contrib agent config (`otelcol-contrib.yaml`) for an EKS DaemonSet that exports metrics, logs, and traces via TLS OTLP to the Gateway NLB endpoint."
- "Generate an OTel Gateway Collector config (`gateway.yaml`) that receives OTLP from agents, applies tail-sampling and PII scrubbing, and exports to Prometheus remote-write, Loki, and Tempo."
- "Write Terraform for an AWS ASG + NLB running otelcol-contrib Gateway nodes, with a target-tracking scaling policy based on CPU."
- "Write a `docker-compose.yml` for local testing with otelcol-contrib (agent + gateway), Prometheus, Loki, Tempo, and a Grafana instance simulating Managed Grafana."
- "Create a phased rollout plan (weeks 1–4, 5–8, 9–12) for deploying OTel Collectors across AWS, on-prem, and DigitalOcean, with Gateway ASG stood up in week 1."
- "Draft a 2–3 paragraph rationale for choosing AWS Managed Grafana + OTel Gateway pattern over Datadog/Splunk for a mixed AWS/on-prem/DO environment."
- "Generate a Grafana dashboard JSON for visualizing OTel Gateway health (queue length, export errors, spans/metrics/logs received and exported, ASG instance count)."
- "Write an ALB listener rule (Terraform) that enforces OIDC auth before forwarding to AWS Managed Grafana, with WAF WebACL attached."

## Verification & Follow-ups
- After generating content, validate: correct OTel Collector component names (receivers, processors, exporters), security posture, credential handling, and cost assumptions.
- Ensure all configs reference `otelcol-contrib` (not the core distribution) since contrib includes the required receivers.
- Ask for a test plan for any generated scripts or IaC.

## Contacts / Notes
- Reference `original.md` for the full exercise constraints and deliverables.

