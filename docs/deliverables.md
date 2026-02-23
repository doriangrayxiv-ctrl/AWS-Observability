# Deliverables

High-level observability architecture document to accompany the Arch_Diagram.png

---

## 1. Architecture Diagram

> High-level architecture only. Fine-grained details (storage sizing, OTel Collector config, IaC) are out of scope for this diagram.

- **Data Sources** — Shopify, Retail POS, Intranet, B2B website, ETL pipelines, scheduled jobs, Snowflake, Postgres/MySQL, EKS, ECS Fargate, Lambda, RDS
- **Collection** — OTel Collector Contrib agents on Linux and Windows hosts; DaemonSet on EKS; sidecar containers on ECS Fargate; Lambda layer for serverless functions
- **Transport** — TLS-encrypted OTLP from agents → Network Load Balancer (NLB) → OTel Gateway ASG; on-prem via VPN/Direct Connect; DigitalOcean via WireGuard tunnel
- **Gateway Processing** — OTel Collector gateway tier on Auto Scaling Group; tail-sampling, PII scrubbing, attribute normalization, metric/span filtering, batching before export
- **Storage Backends** — Prometheus (metrics, TSDB on EBS); Grafana Loki (logs, WAL/index on EBS + chunks in S3); Grafana Tempo (traces, WAL on EBS + blocks in S3)
- **Visualization** — AWS Managed Grafana at `grafana.company.com:443`; unified dashboards across metrics, logs, and traces; self-service for Viewer/Editor roles
- **Access Controls** — AWS WAF (rate-limiting, geo/IP filtering) → ALB with OIDC authentication (TLS termination, IdP-enforced login) → Managed Grafana RBAC (Viewer/Editor/Admin roles, folder-level permissions, team-based assignments, data source access restrictions)

---

## 2. Processing Strategy

Not all processing is deferred to the Gateway. Edge collectors perform a limited set of lightweight operations where it makes sense to do so before data leaves the host:

- **Format normalisation** — some sources emit data in non-OTLP formats (e.g. Prometheus exposition, Windows Event Log XML, syslog). The edge collector transforms these into OTLP before forwarding, so the Gateway and all downstream components deal with a single, consistent wire format.
- **Coarse filtering** — verbose sources (e.g. debug-level application logs, high-cardinality trace spans from noisy background threads) can be filtered at the edge to avoid saturating the transport path. Only a representative subset is forwarded rather than the full volume.
- **Deduplication** — where an agent collects multiple repeated events in a given interval, duplicates are tallied+merged before forwarding.

Edge processing is intentionally minimal — the goal is to keep agent configs simple and host-resource overhead low. No PII handling, tail-sampling, or complex routing logic belongs at the edge.

The **OTel Gateway cluster** (ASG behind the NLB) is the primary processing tier. All telemetry from every source converges here before reaching a storage backend. The Gateway is responsible for:

- **Tail-based trace sampling** — decisions made after the full trace is assembled, not per-span at the edge
- **PII scrubbing** — attribute processors redact or hash sensitive fields before data is written to any store
- **Attribute normalisation** — service names, environment labels, and resource attributes are standardised across all sources
- **Metric and span filtering** — fine-grained drop rules applied centrally rather than duplicated across every agent
- **Routing** — telemetry is fanned out to the correct backend (Prometheus, Loki, or Tempo) based on signal type
- **Batching** — output is batched and compressed for efficient backend ingest

---

## 3. Alerting Strategy

The alerting design follows a **two-path model** that separates concerns by signal fidelity and delivery dependency:

- **Path 1 — AWS-Native (EventBridge + CloudWatch Alarms):** Detects infrastructure and container lifecycle events directly from the AWS control plane. This path operates independently of the observability stack and fires even when the OTel pipeline is degraded or completely down — making it the safety net for pipeline-level failures (ECS sidecar blackout, NLB unhealthy targets, ASG instance floor, disk saturation).
- **Path 2 — Grafana Managed Alerting (AMG):** Evaluates metric and log signal thresholds using Prometheus, Loki, and CloudWatch datasources. This path provides richer, application-aware alerting but depends on the pipeline being healthy — used for OTel Gateway export errors, queue saturation, and backend ingestion health.

Both paths can route notifications through **Amazon SNS → PagerDuty / Slack**, ensuring a single operator notification workflow regardless of which path fires.

---

## 4. Written Rationale (2–3 paragraphs)

### Rationale

A few sizing assumptions anchored this design. At peak I expected roughly **1,000+ concurrent OTel Collector connections** hitting the Gateway, generating around **8 GB of telemetry per day**. On the consumption side I assumed a small team — approximately **10 concurrent Grafana users** — with access segmented by role (Viewer / Editor / Admin) through IAM Identity Center. Working from those numbers, 90-day retention lands at roughly 150 GB for Prometheus, 56 GB for Loki (index on EBS, chunks in S3), and 28 GB for Tempo (WAL on EBS, blocks in S3) — about 234 GB in total. Some assumptions were made for current architecture — for example, whether scheduled scripts emit structured logs or need an SDK import — I called out the ambiguity and designed for the more conservative case rather than assuming the best one.

The stack is entirely open-source and runs within AWS, which was an already established infrastructure platform. The central choice was **OpenTelemetry Collector Contrib** as the sole collection agent. It ships more receivers than any alternative, which means a single binary covers every data source in the estate without bolting on Fluentd, Telegraf, or a separate trace agent. That consolidation matters operationally: one agent means one config surface, one upgrade path, and lower per-host resource overhead. For storage, **Prometheus**, **Loki**, and **Tempo** are each purpose-built for their signal type, carry no per-host licensing, and are well-understood by most DevOps teams. **AWS Managed Grafana** was the natural choice for the visualization layer — it eliminates the need to operate Grafana HA, comes with IAM Identity Center SSO out of the box, and includes the Enterprise RBAC features required for proper multi-team self-service. Datadog and New Relic were ruled out early: at 1,000+ collector-scale their per-host and per-volume pricing reaches an unfavorable cost, and the vendor lock-in works against the open-standards direction the platform is built on. CloudWatch stays in the picture, but specifically as an independent alerting safety net rather than a primary telemetry store.

The three signal types — metrics, logs, and traces — each take a slightly different path to full coverage, which is intentional. Metrics and logs flow from day one with minimal or no application changes: OTel Collector agents read what the infrastructure already produces and forward it through the Gateway. Traces are a different story — they require at least a small instrumentation step per service, so they roll out incrementally. The important point is that a service without trace coverage is not invisible: it still contributes full metric and log data from the moment the agent is deployed. The platform delivers immediate, broad visibility on day one and deepens as trace instrumentation is added service by service.

---

## 5. Rollout Plan / Project Phasing

> **Resource assumption:** A single engineer owns all aspects — coordination, implementation, pipeline builds, and stakeholder communication. 

It's worth being honest about the timeline: 4 weeks to value is optimistic for one engineer carrying this end-to-end. It assumes no interruptions, no competing priorities, no procurement delays (VPN provisioning, IdP onboarding, Direct Connect lead times), and no surprises in the existing infrastructure. In practice, a more realistic delivery window for a solo implementer is **5 weeks** for value ramp up and closer to **16–20 weeks** for completion when accounting for coordination overhead across multiple teams, waiting on approvals, and the debugging that comes with connecting systems that have never connected in this manner. The phasing below reflects the most efficient sequential order — not a guarantee of pace.

The first two weeks are entirely setup — confirming access, standing up distribution pipelines, and running a proof-of-concept to validate the pipeline before committing to production infrastructure. Weeks 3 and 4 are backend infrastructure: the Gateway, storage backends, and Grafana. None of this produces dashboards anyone can use yet. **The first real, visible value arrives at Week 5**, when agents are deployed to AWS hosts and real metrics and logs start flowing into Grafana for the first time.


| Phase | Week | Focus |
|-------|-----:|-------|
| 1 — Foundation | 1 | Prerequisites: SSM/Ansible distribution pipeline, CI/CD access (EKS/ECS/Lambda), S3 artifact bucket, TLS certs (ACM), IAM roles, notification tokens |
| 1 — Foundation | 2 | POC / dev validation: end-to-end pipeline on a single EC2, TLS + OIDC auth walkthrough, stakeholder demo and sign-off |
| 1 — Foundation | 3–4 | Production backend infra: Gateway NLB + ASG, Prometheus, Loki, Tempo, AWS Managed Grafana (WAF + ALB + OIDC) |
| 1 — Foundation | 5 | First AWS data sources: Linux hosts, RDS receivers, EKS DaemonSet, ECS Fargate sidecars — zero application code changes |
| 1 — Foundation | 6 | First dashboards (host, DB, container, pipeline health) and safety-net alerting (both CloudWatch and Grafana paths live) |
| 2 — Expanded coverage | 7 | Remote connectivity: DigitalOcean WireGuard tunnel + on-prem VPN/Direct Connect; agent deployment to remote hosts |
| 2 — Expanded coverage | 8 | SaaS and API sources: Shopify webhook/polling, POS API polling, Snowflake audit API |
| 2 — Expanded coverage | 9 | Lambda OTel layer + first trace rollout on one high-value service (auto-instrumentation) |
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
| Region | us-west-2 |
| Instance sizing | Minimum viable |

### Signal Distribution

| Signal | Split | Daily Volume | 90-Day Total |
|---|---|---|---|
| **Logs** | 70% | 5.6 GB/day | 504 GB |
| **Metrics** | 20% | 1.6 GB/day | 144 GB |
| **Traces** | 10% | 0.8 GB/day | 72 GB |
| **Total** | 100% | 8 GB/day | 720 GB |

### Monthly Cost (us-west-2)

| Component | Resource | Unit Cost | Monthly Cost |
|---|---|---|---|
| **OTel Gateway ASG** | 2× `t3.medium` On-Demand | $0.0416/hr each | $61 |
| **Gateway NLB** | 1× NLB + LCU | ~$0.006/LCU-hr | $22 |
| **Prometheus EC2** | 1× `t3.large` | $0.0832/hr | $61 |
| **Prometheus EBS** | 150 GB gp3 | $0.08/GB/mo | $12 |
| **Loki EC2** | 1× `t3.medium` | $0.0416/hr | $30 |
| **Loki S3 Storage** | 504 GB | $0.023/GB/mo | $12 |
| **Loki S3 Requests** | PUT/GET ops | Estimated | $5 |
| **Tempo EC2** | 1× `t3.small` | $0.0208/hr | $15 |
| **Tempo S3 Storage** | 72 GB | $0.023/GB/mo | $2 |
| **Tempo S3 Requests** | PUT/GET ops | Estimated | $2 |
| **AWS Managed Grafana** | 10 editors | $9/editor/mo | $90 |
| **ALB (Grafana access)** | 1× ALB + LCU | ~$0.008/LCU-hr | $18 |
| **AWS WAF** | 1 WebACL + 5 rules | $5 WebACL + $1/rule/mo | $10 |
| **VPN Site-to-Site** | 2× connections | $0.05/hr each | $72 |
| **Data Transfer** | Inter-AZ + egress | Variable | $15 |
| **CloudWatch / Secrets** | RDS metrics + credentials | Minimal | $5 |
| **Total** | | | **~$432/mo** |

> Committing to 1-year Reserved Instances on Prometheus, Loki, and Tempo EC2 saves approximately 35% on those compute lines (~$40/mo). S3 storage stabilises at 90-day rolling retention once deletion lifecycle rules are applied.
