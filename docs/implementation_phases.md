# Implementation Phases

Phased rollout plan for the centralized observability platform.

> **Resource assumption:** A single engineer owns all aspects of this project — stakeholder coordination, infrastructure provisioning, pipeline builds, agent deployments, instrumentation, and ongoing tuning. All phases are **sequential**. No parallel workstreams are assumed.

---

## Phase 1 — Foundation & First Signal (Weeks 1–6)

Phase 1 establishes the entire foundation: distribution pipelines, core backend infrastructure, all AWS-hosted data sources, and the first dashboards and alerts. Every subsequent phase depends on this foundation being stable and trusted.

### Week 1: Prerequisites & Distribution Pipelines

Confirm or stand up the mechanisms needed to distribute the OTel Collector binary and configuration to every host type in the estate. Most items here are confirmations, not builds — with full access, the engineer resolves these in days, not weeks.

### Config Distribution for Host Agents

Assess what exists today and establish the minimum viable mechanism:

- **AWS hosts (Linux and Windows):** Prefer **AWS Systems Manager (SSM) Distributor** if SSM is already in the estate — it requires no new tooling, distributes packages to EC2 instances via IAM role, and handles both Linux and Windows. If SSM is not available or on-prem hosts are out of SSM scope, stand up a minimal **Ansible** control node and confirm SSH/WinRM connectivity to all target hosts.
- **On-prem and DigitalOcean hosts:** These are outside AWS and cannot use SSM Distributor. Confirm Ansible reach over the VPN/WireGuard path established in Phase 2. If VPN is not yet in place, Ansible playbooks for remote hosts are written in Phase 2 Week 7 alongside the tunnel work.
- **Collector binary and config storage:** Create an S3 bucket (or confirmed artifact store) to host `otelcol-contrib` binaries and versioned config files. Distribution scripts pull from this location. This decouples the distribution mechanism from the binary source and enables config-only updates without re-deploying the binary.

### CI/CD for Container and Kubernetes Deployments

- **EKS:** Confirm `kubectl` / Helm access to each cluster from the engineer's workstation or a CI runner. The OTel DaemonSet will be deployed via Helm chart (`opentelemetry-helm-charts/opentelemetry-collector`); ensure the chart registry is reachable.
- **ECS Fargate:** Confirm access to update ECS task definitions (via AWS CLI, Terraform, or the console). The sidecar pattern requires a task definition update per service; know the deployment mechanism before first data source deployment in Week 5.
- **Lambda:** Confirm access to update Lambda function configurations to attach the OTel layer ARN.

### TLS Certificates

- Confirm a wildcard or SAN certificate covers `nlb.company.com` and `grafana.company.com`, or plan to issue one via ACM before infrastructure work begins in Week 3.
- Confirm the certificate is accessible to the NLB listener and the ALB listener.

### Access and Permissions

- Confirm IAM roles and policies exist (or are ready to be created) for: EC2 instances hosting Gateway/backends (SSM, CloudWatch, S3), AMG workspace (datasource IAM roles), and `awscloudwatchreceiver` scraping.
- Confirm IAM Identity Center is provisioned and an IdP (or the AWS SSO directory) is available for Grafana authentication.
- Confirm PagerDuty and Slack integration tokens are available for the SNS subscription step in Phase 1 Week 6.

*Exit criterion: all items above are confirmed in writing; no ambiguous dependencies remain before Week 2 POC work begins.*

---

### Week 2: Proof of Concept & Dev Validation

Before committing production sizing and fleet-wide rollout, validate the full pipeline end-to-end in a minimal, isolated environment. This phase de-risks the production deployment and gives stakeholders an early Grafana demo to align on expectations. A single EC2 with Docker Compose gets this done in days.

**Minimal dev environment**
- Spin up a single EC2 instance running all components locally: `otelcol-contrib` (Gateway mode), Prometheus, Loki, and Tempo — no NLB, no ASG, no HA. The goal is pipeline correctness, not production architecture.
- Use Docker Compose or a simple systemd setup; tear down after this phase.
- Confirm all three backends accept ingest from the Collector and are queryable in Grafana (using the production AMG workspace pointed at dev endpoints, or a temporary local Grafana instance).

**One of each source type**
- Deploy one `otelcol-contrib` agent on a single non-critical Linux host; confirm metrics and logs flow end-to-end to Prometheus and Loki.
- Run `postgresqlreceiver` or `mysqlreceiver` against a non-production (or read replica) database instance; confirm metrics land in Prometheus.
- Deploy the ECS sidecar pattern in a non-production task definition; confirm container telemetry flows to the dev pipeline.

**TLS and auth validation**
- Validate the full NLB TLS pass-through path using the actual ACM certificate in a test listener before production provisioning.
- Walk through the OIDC login flow on the ALB + AMG to confirm the IdP redirect, token exchange, and Grafana role assignment work as expected.

**Stakeholder demo**
- Present at least one working Grafana dashboard with live data to a stakeholder before production rollout begins. Confirm alignment on what the finished dashboards should show.

*Exit criterion: data flows from agent → Gateway → all three backends → Grafana with no gaps; TLS and OIDC auth validated; stakeholder sign-off on dashboard direction received.*

---

### Week 3–4: Backend Infrastructure

Stand up the production storage backends and the Gateway tier. This is the most infrastructure-heavy block in the project — NLB + ASG + TLS, three backend servers, and AMG + WAF + ALB + OIDC all need to come online. Two weeks accounts for inevitable networking and security group debugging.

**OTel Gateway (NLB + ASG)**
- Provision the AWS Network Load Balancer (`nlb.company.com:443`); configure TCP/TLS pass-through listeners on port 443 mapped to OTLP gRPC (4317) and HTTP (4318) on Gateway instances.
- Launch the OTel Gateway ASG with `otelcol-contrib`; configure a minimal pass-through pipeline first (receive → batch → export) to validate end-to-end connectivity before enabling enrichment and sampling rules.
- Validate TLS certificate chain on the NLB; confirm agents can connect and data reaches the backends.
- Configure ASG scaling policy on OTel Collector queue depth and CPU.

**Prometheus**
- Deploy Prometheus on an EC2 instance with an EBS volume sized for the 90-day TSDB target (~150 GB).
- Confirm remote-write endpoint is reachable from the Gateway.

**Grafana Loki**
- Deploy Loki in simple-scalable or monolithic mode; configure EBS for WAL and index (~20 GB), S3 bucket for chunk storage (~36 GB target at 90 days).
- Confirm Loki OTLP/HTTP ingest endpoint is reachable from the Gateway.

**Grafana Tempo**
- Deploy Tempo; configure EBS for WAL (~10 GB), S3 bucket for trace blocks (~18 GB target at 90 days).
- Confirm Tempo OTLP ingest endpoint is reachable from the Gateway.

**AWS Managed Grafana**
- Provision the AMG workspace; connect IAM Identity Center for SSO.
- Configure WAF with rate-limiting and geo/IP rules on `grafana.company.com:443`.
- Configure ALB with TLS termination and OIDC authentication rule; unauthenticated requests return 401/redirect.
- Add Prometheus, Loki, and Tempo as datasources in Grafana; validate all three with test queries.

*Exit criterion: `otelcol-contrib` test agent → NLB → Gateway → backends → Grafana query returns data.*

---

### Week 5: First Data Sources (AWS-Hosted, Zero Code Changes)

Deploy OTel Collector agents to all AWS-hosted infrastructure using the distribution pipeline built in Week 1. The agent config is a near-identical template per host — this is repetitive distribution work, not engineering. These sources require no application changes.

**AWS Linux Hosts (Intranet, B2B, ETL pipelines, scheduled jobs)**
- Install `otelcol-contrib` as a systemd service on each Linux host.
- Enable `hostmetricsreceiver` (CPU, memory, disk, network) and `filelogreceiver` for application log paths.
- Configure each agent to export TLS OTLP to `nlb.company.com:443`.
- Apply resource detection processor (`resourcedetectionprocessor`) to attach `host.name`, `cloud.region`, and service identity attributes.

**AWS RDS (Postgres and MySQL)**
- Enable `postgresqlreceiver` and `mysqlreceiver` in the Gateway or a dedicated scraper Collector instance.
- Enable `awscloudwatchreceiver` for RDS CloudWatch metrics (FreeStorageSpace, DatabaseConnections, ReadLatency, WriteLatency).

**AWS EKS**
- Deploy `otelcol-contrib` as a DaemonSet; configure `kubeletstatsreceiver` for pod/node metrics and `filelogreceiver` for container logs via the node's `/var/log/pods` path.
- Annotate namespaces to enable OTel auto-instrumentation where supported.

**AWS ECS Fargate**
- Add `otelcol-contrib` as a sidecar container (`essential: false`) in each task definition.
- Configure the sidecar to receive OTLP from the application container on localhost and forward to the Gateway NLB.

*Exit criterion: host metrics, database metrics, and container logs are visible in Grafana for all AWS-hosted systems.*

---

### Week 6: First Dashboards and Alerts

With data flowing, stand up the minimum viable dashboards and safety-net alerting. Dashboards draw from community templates; alert rules use the PromQL queries already defined in the alerting design.

**Dashboards**
- Host health: CPU, memory, disk utilisation, and system load — one dashboard per environment tag.
- Database health: connections, query latency, replication lag, storage headroom — for RDS Postgres and MySQL.
- EKS / ECS container health: pod/task status, restarts, CPU and memory per workload.
- OTel Pipeline health: Gateway queue depth, export error rate, dropped spans/metrics/logs, NLB target health.

**AWS-Native Safety-Net Alerts (Path 1)**
- CloudWatch Alarms: RDS disk saturation, ECS sidecar container stopped (A-01), NLB unhealthy targets (A-04), EBS utilisation on backend nodes (A-05), ASG instance count below floor (A-06).
- EventBridge rule for ECS Task State Change → dead sidecar detection (A-01).
- Route all alarms through SNS → PagerDuty / Slack.

**Grafana Managed Alerts (Path 2)**
- OTel Gateway export errors (A-02), queue saturation (A-03).
- Prometheus ingestion health (A-07), Loki ingestion health (A-08), Tempo ingestion health (A-09).
- Route through SNS → PagerDuty / Slack.

*Exit criterion: all Phase 1 alerts are firing correctly in test conditions; on-call rotation receives notifications via both paths.*

---

## Phase 2 — Expanded Coverage (Weeks 7–9)

The platform is proven. Expand to remaining infrastructure, connect external data sources, and begin trace rollout.

### Week 7: Remote & On-Prem Connectivity

Agent deployment reuses the same SSM/Ansible pattern and config templates from Phase 1. The main effort is establishing the network paths — VPN/WireGuard tunnel setup and routing validation.

**DigitalOcean Hosts (scheduled jobs)**
- Establish site-to-site WireGuard tunnel between DigitalOcean and the AWS VPC.
- Deploy `otelcol-contrib` on each DO host; configure `hostmetricsreceiver` and `filelogreceiver`; export TLS OTLP through the tunnel to `nlb.company.com:443`.

**On-Premises Hosts (databases, Windows hosts)**
- Confirm VPN or Direct Connect path to the Gateway NLB is in place.
- Deploy `otelcol-contrib` on Linux on-prem database hosts using the same systemd pattern as AWS Linux hosts.
- Deploy `otelcol-contrib` as a Windows service on Windows hosts; configure `hostmetricsreceiver` with Windows-specific performance counter targets and `windowseventlogreceiver` for Windows Event Log.
- Enable `postgresqlreceiver` / `mysqlreceiver` for on-prem database instances.

### Week 8: SaaS and API Sources

Each source follows the same pattern: HTTP polling or webhook receiver → OTLP → Gateway. The integration work per source is 1–2 days including its dashboard.

**Shopify (Ecommerce)**
- Configure a webhook receiver in the Gateway (or a lightweight OTel Collector HTTP receiver) to accept Shopify order and fulfilment events.
- Alternatively, poll the Shopify Admin API on a schedule and push metrics into the pipeline via an OTLP exporter in a small Python script.
- Produce a Grafana dashboard for order volume, fulfilment rate, and checkout error rate.

**POS (Retail, REST API)**
- Configure API polling in the Gateway or a dedicated Collector; extract transaction counts and error rates.
- Add a POS health dashboard alongside the ecommerce dashboard.

**Snowflake (Data Warehouse)**
- Poll Snowflake usage and audit query APIs; extract credit consumption, query duration, and warehouse utilisation metrics.
- Forward via OTLP to the Gateway; build a Snowflake cost and performance dashboard.

### Week 9: AWS Lambda and Initial Trace Rollout

Lambda layer attachment is config-only (no code change). The first trace rollout on a single service is the more significant effort — auto-instrumentation keeps it to 2–3 days.

**Lambda**
- Add the OTel Lambda layer to each function via Lambda configuration (no code change required for auto-instrumentation in supported runtimes).
- Validate that traces, metrics, and logs flow from Lambda through the Gateway to Tempo and Loki.

**First Trace Rollout (one service)**
- Choose the highest-value, lowest-risk service as the first trace target — recommended: B2B website or Intranet (JS/PHP, running on an already-instrumented AWS host).
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
- **ECS Fargate services:** Instrument application containers to emit OTLP traces to the sidecar Collector on localhost.
- **EKS workloads:** Apply the OTel Operator and `Instrumentation` CRD for language-level auto-instrumentation where supported; manual SDK for remaining services.
- For each service with traces enabled, validate trace-to-log and trace-to-metric correlation in Grafana Explore.

### Week 12: Alerting Maturity

Technical implementation — recording rules, alert rules, PagerDuty routing — is 3–4 days. The variable is stakeholder alignment on SLO targets, but the engineer drives those conversations in parallel with the technical work.

- **SLO-based alerting:** Define SLOs for key services (e.g. B2B checkout availability ≥ 99.5%, API p99 latency < 500 ms). Implement Grafana recording rules and alerting rules that evaluate against these targets.
- **Runbook links:** Attach a `runbook_url` annotation to every alert rule, pointing to an internal runbook in the company wiki.
- **Escalation tiers in PagerDuty:** Configure severity-based routing — Critical alerts page on-call immediately; Warning alerts enter a low-urgency queue that notifies Slack only during business hours.
- **Alert deduplication:** Review firing alert volume; suppress noisy derived alerts that duplicate upstream root-cause alerts.

### Week 13: RBAC Rollout + Cost Review and Retention Tuning

Two smaller workstreams combined into one week. RBAC is 2–3 days of configuration (OIDC claim mapping was already validated in the POC). Cost review uses ~10 weeks of observed data to extrapolate 90-day storage and adjust sizing.

**RBAC (first half of week)**

- Map OIDC group claims from IAM Identity Center to Grafana workspace roles: `Viewer` (all staff), `Editor` (DevOps / SRE), `Admin` (platform owner only).
- Create team-scoped dashboard folders: Ops team sees infrastructure dashboards; Dev teams see application/service dashboards; Finance sees cost dashboards.
- Restrict direct Explore access to Prometheus, Loki, and Tempo datasources to `Editor+` roles.
- Validate that a Viewer-role test account cannot access restricted datasources or folders.

**Cost Review and Retention Tuning (second half of week)**

- Review actual storage consumption against Phase 1 estimates (150 GB Prometheus, 56 GB Loki, 28 GB Tempo at 90 days).
- Adjust Prometheus TSDB `--storage.tsdb.retention.time` and block compaction settings if consumption is trending above estimate.
- Review Loki chunk compaction schedule; move chunks older than 30 days to S3 Infrequent Access storage class.
- Review Tempo block TTL; move cold blocks to S3 Glacier Instant Retrieval if read frequency is low.
- Review OTel Gateway ASG sizing: if p95 CPU is consistently below 40%, reduce the minimum instance count.
- Produce a one-page cost summary: actual monthly spend on EC2 (Gateway + backends), EBS, S3, AMG, NLB, and data transfer versus initial estimates.

*Exit criterion: all data sources covered, full trace rollout complete, SLO alerting live, RBAC enforced, 90-day retention validated against budget.*

---

## Summary Timeline

| Phase | Weeks | Focus |
|-------|-------|-------|
| 1 | 1 | Prerequisites: distribution pipelines (SSM/Ansible), CI/CD access, TLS certs, IAM roles, notification tokens |
| 1 | 2 | Proof of concept: end-to-end pipeline validation in dev, TLS/auth walkthrough, stakeholder demo |
| 1 | 3–4 | Core backend infrastructure (Gateway NLB+ASG, Prometheus, Loki, Tempo, AMG+WAF+ALB+OIDC) |
| 1 | 5 | First data sources: all AWS-hosted hosts, RDS, EKS, ECS Fargate |
| 1 | 6 | First dashboards and safety-net alerting (both paths live) |
| 2 | 7 | Remote connectivity: DigitalOcean WireGuard + on-prem VPN/Direct Connect + host agents |
| 2 | 8 | SaaS and API sources: Shopify, POS, Snowflake |
| 2 | 9 | Lambda layer + first trace rollout (one service) |
| 3 | 10–11 | Complete trace rollout across all services |
| 3 | 12 | Alerting maturity: SLOs, runbooks, escalation tiers |
| 3 | 13 | RBAC rollout + cost review, retention tuning, handover documentation |
