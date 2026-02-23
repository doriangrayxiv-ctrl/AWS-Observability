# Presentation Layer — Grafana Platform

## Overview

AWS Managed Grafana (AMG) serves as the single pane of glass for dashboards, alerting, and exploration. Dashboard users access Grafana at `grafana.company.com:443` through a WAF → ALB (OIDC) → AMG workspace access path; only authenticated sessions reach the AMG workspace, where Grafana RBAC then controls what each user can see and do. Backend datastores (Prometheus, Loki, Tempo) run on EC2 in private subnets and receive telemetry pushed from authenticated sources across multiple AWS regions and on-prem environments.

---

## AWS Resources

| Resource | Purpose | Notes |
|----------|---------|-------|
| **NLB** (Network Load Balancer) | TLS termination for high-volume telemetry push traffic | Fronts OTel Gateway fleet; 1000+ endpoint scale |
| **AWS WAF** | Inspects and filters inbound HTTPS requests to Grafana | Rate-limiting, geo/IP rules; attached to ALB |
| **ALB** (Application Load Balancer) | HTTPS termination for Grafana dashboard users; enforces OIDC | Unauthenticated requests receive 401/redirect |
| **AWS Managed Grafana (AMG)** | Managed Grafana workspace; receives only authenticated requests from ALB; enforces Grafana RBAC post-authentication | `grafana.company.com:443`; IAM Identity Center or SAML/OIDC SSO; Viewer / Editor / Admin roles mapped from IdP claims |
| **OTel Collector Gateway (ASG)** | Receive, buffer, batch, route telemetry to backends | 2–4 instances (t3.medium), Auto Scaling Group |
| **ACM Certificate** | TLS certs for NLB + ALB HTTPS listeners | Auto-renewing via AWS Certificate Manager |
| **EC2 Instances** | Run Prometheus, Loki, Tempo | Sized per retention/volume needs; no EC2 for Grafana |
| **EBS Volumes (gp3)** | Hot storage: Prometheus TSDB, Loki WAL + index, Tempo WAL | ~180 GB total; sized for active working set only |
| **S3 Bucket** | Bulk/cold storage: Loki chunks, Tempo trace blocks | ~54 GB at 90-day retention; multi-AZ durable; Prometheus does not use S3 |
| **Security Groups** | Control inbound/outbound traffic per service | See port table below |
| **VPC** | Isolated network for the observability stack | Private subnets for backends, public subnet for ALB |
| **VPC Peering / TGW** | Cross-region connectivity from other AWS regions | Peering or Transit Gateway per region |
| **Site-to-Site VPN** | On-prem → VPC connectivity | IPsec or WireGuard tunnel |
| **Route 53** | DNS for `grafana.company.internal` / public FQDN | Alias to ALB |
| **IAM Roles** | Service-level permissions, ALB access logs | Least-privilege per component |

---

## Ports & Protocols

Two separate traffic profiles flow through this architecture: high-volume machine push (NLB path) and low-volume human dashboard access (ALB path). The table is grouped by flow type for clarity.

### External Entry Points

| Port | Protocol | Direction | Listener | Caller (source) |
|------|----------|-----------|----------|-----------------|
| **443** | HTTPS/gRPC | Inbound | NLB → OTel Gateway fleet | OTel Collectors (all sources — AWS, on-prem, DigitalOcean); OTLP push |
| **443** | HTTPS | Inbound | WAF → ALB → AMG workspace | Dashboard users (browser); OIDC enforced at ALB |

### Internal — OTel Gateway Receivers (NLB forwards inbound :443 collector traffic here)

| Port | Protocol | Direction | Listener | Caller (source) |
|------|----------|-----------|----------|-----------------|
| **4317** | gRPC | Internal | OTel Gateway (OTLP gRPC receiver) | NLB; forwards all inbound collector OTLP gRPC traffic |
| **4318** | HTTP | Internal | OTel Gateway (OTLP HTTP receiver) | NLB; forwards all inbound collector OTLP HTTP traffic |

### Internal — OTel Gateway → Backends (Gateway is the sole writer; no source writes directly to a backend)

| Port | Protocol | Direction | Listener | Caller (source) |
|------|----------|-----------|----------|-----------------|
| **9090/api/v1/write** | HTTP | Internal | Prometheus (remote-write ingest) | OTel Gateway (Prometheus remote-write exporter; metrics only) |
| **3100/loki/api/v1/push** | HTTP | Internal | Loki (log push ingest) | OTel Gateway (Loki exporter; logs only) |
| **4317** | gRPC | Internal | Tempo (OTLP trace ingest) | OTel Gateway (OTLP exporter; traces only) |

### Internal — AMG → Backends (read-only datasource queries)

| Port | Protocol | Direction | Listener | Caller (source) |
|------|----------|-----------|----------|-----------------|
| **9090** | HTTP | Internal | Prometheus (query API) | AWS Managed Grafana (datasource queries) |
| **3100** | HTTP | Internal | Loki (query API) | AWS Managed Grafana (datasource queries) |
| **3200** | HTTP | Internal | Tempo (query API) | AWS Managed Grafana (datasource queries) |

> **All internal traffic** stays within the VPC on private subnets. All backend ports are unreachable from outside the VPC.  
> **The NLB is the sole external entry point for telemetry.** It forwards to the OTel Gateway fleet, which is the sole writer to all backends (Prometheus, Loki, Tempo). No collector or external source writes to a backend directly.  
> **The ALB is the sole external entry point for dashboard users.** It carries read-only datasource queries only and is never in the telemetry push path.

---

## Authentication & Authorization

| Flow | Method |
|------|--------|
| Dashboard users → Grafana | HTTPS 443 — OIDC session enforced at ALB before request reaches AMG; AMG workspace uses IAM Identity Center (or SAML) for identity |
| OTel Collectors → Gateway NLB | HTTPS 443 — `bearertokenauth` extension in OTel Collector config, or mTLS at the NLB TLS listener; ALB is not in the push path |

### Grafana RBAC (post-authentication authorization)

Once a user's identity is confirmed by the ALB OIDC layer, AWS Managed Grafana applies fine-grained RBAC (Grafana Enterprise, included in AMG) to control what they can see and do:

| Control | Behaviour |
|---------|-----------|
| **Workspace roles** | OIDC claims from the IdP are mapped to Grafana **Viewer / Editor / Admin** workspace roles |
| **Folder-level permissions** | Restrict which teams can access which dashboards (e.g. Ops team → infra dashboards; Dev team → application dashboards) |
| **Data source permissions** | Control which roles can query Prometheus, Loki, or Tempo via Explore — limited to Editor+ by default |
| **Team-based assignments** | Users are assigned to teams aligned to IdP groups; permissions are granted at the team level for bulk management |

This creates a two-step access model: the **ALB enforces authentication** (no unauthenticated request reaches AMG), and **Grafana RBAC enforces authorization** (authenticated users see only what their role permits).

---

## ASCII Architecture Map

```
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │                         EXTERNAL SOURCES                                    │
 │                                                                              │
 │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
 │   │ AWS Region A │  │ AWS Region B │  │  On-Prem DC  │  │ DigitalOcean │   │
 │   │  OTel Coll.  │  │  OTel Coll.  │  │  OTel Coll.  │  │  OTel Coll.  │   │
 │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
 │          │                  │                  │                  │           │
 │          │  OTLP :443       │  OTLP :443       │  VPN + :443     │  VPN+:443 │
 └──────────┼──────────────────┼──────────────────┼──────────────────┼───────────┘
            │                  │                  │                  │
            ▼                  ▼                  ▼                  ▼
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │                      AWS — Observability VPC                                │
 │                                                                              │
 │  PUSH PATH (high volume — 1000+ sources)                                    │
 │   ┌──────────────────────────────────────┐                                  │
 │   │         NLB  (TLS :443)              │  ◄── low cost, high throughput   │
 │   │     ACM Cert · TCP passthrough       │                                  │
 │   └──────────┬──────────┬────────────────┘                                  │
 │              │          │          │                                         │
 │              ▼          ▼          ▼                                         │
 │        ┌──────────┐┌──────────┐┌──────────┐                                 │
 │        │  OTel GW ││  OTel GW ││  OTel GW │  ◄── ASG, gateway mode         │
 │        │  :4317   ││  :4317   ││  :4317   │     buffer / batch / route      │
 │        └────┬─────┘└────┬─────┘└────┬─────┘                                 │
 │             └─────┬─────┴─────┬─────┘                                       │
 │                   │           │                                              │
 │          ┌────────┴──┐  ┌─────┴──────┐  ┌──────────┐                        │
 │          │Prometheus │  │    Loki    │  │  Tempo   │   Private Subnets      │
 │          │   :9090   │  │   :3100    │  │:3200/4317│                        │
 │          └─────┬─────┘  └─────┬──────┘  └────┬─────┘                        │
 │                │              │               │                              │
 │                │   datasource queries         │                              │
 │                └──────┬───────┴───────────────┘                              │
 │                       │                                                      │
 │  QUERY PATH (low volume — dashboard users)                                  │
 │   ┌──────────────────────────────────────┐                                  │
 │   │        ALB  (HTTPS :443)             │  ◄── L7 features, WAF, OIDC     │
 │   │    ACM Cert · Path routing           │                                  │
 │   └──────────────────┬───────────────────┘                                  │
 │                      │                                                       │
                 ┌──────────────────────┐                                    │
                 │  AWS Managed Grafana  │                                    │
                 │  (AMG workspace)      │                                    │
                 └──────────────────────┘                                    │
 │                                                                              │
 │   ┌──────────────────────────────────────────────────────────────┐          │
 │   │  Persistent Storage (90-day retention)                       │          │
 │   │                                                              │          │
 │   │  EBS gp3 (hot / WAL)             S3 (bulk, 90-day)          │          │
 │   │  Prometheus TSDB  ~150 GB        Loki chunks    ~36 GB      │          │
 │   │  Loki WAL + index  ~20 GB        Tempo blocks   ~18 GB      │          │
 │   │  Tempo WAL         ~10 GB        (Prometheus: EBS only)     │          │
 │   └──────────────────────────────────────────────────────────────┘          │
 │                                                                              │
│   Dashboard Users ─── HTTPS :443 ──→ WAF → ALB (OIDC) ──→ AMG workspace   │
 │   1000+ Collectors ── OTLP :443 ──→ NLB ──→ OTel GW ──→ backends          │
 └──────────────────────────────────────────────────────────────────────────────┘
```

---

## Ingress Strategy — Choosing the Right Load Balancer for 1000+ Push Sources

With 1000+ OTel Collectors continuously pushing metrics, logs, and traces, the ingress layer is the highest-traffic component in the stack. The table below compares viable approaches.

### Option Comparison

#### Option A: ALB (Application Load Balancer) — Original Proposal

| Aspect | Detail |
|--------|--------|
| **How it works** | Layer 7 LB; terminates TLS, inspects HTTP paths, routes to target groups |
| **Path routing** | Native — route `/api/v1/write`, `/loki/api/v1/push`, `/v1/traces` to different backends |
| **gRPC support** | Yes (HTTP/2 target groups) |
| **Auth** | Header inspection, OIDC integration, WAF attachment |
| **Scaling** | Auto-scales, but pre-warming needed for sudden spikes |

| Pros | Cons |
|------|------|
| Rich L7 features (path routing, WAF, OIDC) | **Cost scales with LCUs** — 1000+ persistent connections with high throughput drives LCU hours up fast |
| Native ACM TLS termination | Each LCU costs ~$6.50/mo; at sustained load expect 20-50+ LCUs → **$130–$325+/mo just for LB** |
| Easy to operate, fully managed | Higher latency than L4 — full HTTP parse on every request |
| Good for dashboard users (low volume) | Per-request overhead is wasted when all push traffic goes to the same backend tier |

**Estimated cost at scale:** $150–$400/mo (LB only, based on sustained 1000+ connections and high processed-bytes)

---

#### Option B: NLB (Network Load Balancer) — Recommended for Push Traffic

| Aspect | Detail |
|--------|--------|
| **How it works** | Layer 4 LB; passes TCP/TLS streams directly to targets, no HTTP inspection |
| **Path routing** | None — routes by port/protocol only; use separate listeners per backend |
| **gRPC support** | Yes (pass-through; TLS terminated at target or NLB TLS listener) |
| **Auth** | No header inspection — auth must happen at the application layer |
| **Scaling** | Handles millions of connections; designed for high-throughput, low-latency |

| Pros | Cons |
|------|------|
| **Dramatically cheaper** — NLCU pricing favors high-connection, high-throughput workloads; ~60–70% less than ALB at this scale | No path-based routing — need separate ports or a gateway tier behind it |
| Sub-millisecond latency (no L7 parsing) | No WAF attachment |
| Static IPs / Elastic IPs available — easier for on-prem firewall rules | TLS termination options: passthrough or NLB-terminated (no ACM SNI routing) |
| Handles millions of concurrent flows natively | Auth pushed to application or gateway layer |
| Supports TCP, UDP, TLS listeners | Less visibility in access logs vs ALB |

**Estimated cost at scale:** $50–$120/mo (LB only)

---

#### Option C: NLB + OTel Collector Gateway Tier — Best Fit ✓

| Aspect | Detail |
|--------|--------|
| **How it works** | NLB fronts a horizontally-scaled fleet of OTel Collectors in **gateway mode**; gateway collectors receive all push traffic on OTLP :4317/:4318, then route/fan-out to Prometheus, Loki, Tempo |
| **Path routing** | Not needed — the OTel gateway pipeline handles routing via exporters |
| **gRPC support** | Native OTLP gRPC at the collector |
| **Auth** | `bearertokenauth` extension in OTel Collector, or mTLS at NLB |
| **Scaling** | NLB + Auto Scaling Group of gateway collectors; scale on CPU/network |

| Pros | Cons |
|------|------|
| **Cheapest at scale** — NLB cost + small EC2 instances for gateway | Extra compute layer (gateway fleet), but these are lightweight |
| **Buffering & back-pressure** — gateway collectors queue, batch, retry; protects backends from burst spikes | More moving parts to operate |
| **Unified OTLP ingress** — all sources push to one OTLP endpoint; gateway routes metrics→Prometheus, logs→Loki, traces→Tempo | Requires OTel Collector config management for gateway tier |
| **Processing at the edge** — filtering, sampling, attribute enrichment happen in the gateway before data hits storage | Must monitor gateway health (but OTel Collector exports its own metrics) |
| **Protocol normalization** — accepts OTLP gRPC, OTLP HTTP, Prometheus remote-write all on one endpoint | |
| **Decouples sources from backends** — backend migration/scaling is invisible to push sources | |

**Estimated cost at scale:** NLB $50–$120/mo + 2–4 gateway instances (t3.medium ~$30/ea) = **$110–$240/mo total**

---

#### Alternatives Considered

- **Option D — API Gateway (REST):** Rejected. At 1000+ endpoints pushing every 15 seconds, request volume reaches ~170M/month, costing ~$600/mo in API call charges alone. API Gateway has a 30-second timeout and a 10 MB payload limit, neither suited to streaming OTLP/gRPC telemetry workloads.
- **Option E — Self-Managed Reverse Proxy (Nginx/Envoy on EC2):** Rejected. Viable in isolation but operationally redundant — it duplicates the routing and protocol-normalization work that the OTel Collector gateway already handles, while adding a separate HA/scaling/patching burden without the buffering and processing benefits the gateway tier provides.

---

### Recommendation: Split Architecture (NLB + OTel Gateway + ALB)

Separate the two traffic profiles since they have fundamentally different characteristics:

| Traffic Profile | Volume | LB | Target |
|----------------|--------|-----|--------|
| **Dashboard users** (low-volume, human) | Low — tens of concurrent users | **ALB** on :443 | Grafana :3000 |
| **Telemetry push** (high-volume, machine) | High — 1000+ persistent pushers | **NLB** on :443 | OTel Gateway fleet :4317/:4318 |

```
                    1000+ OTel Collectors
                    (AWS, on-prem, DO)
                            │
                     OTLP HTTPS :443
                            │
                            ▼
                 ┌─────────────────────┐
                 │    NLB (TCP/TLS)    │  ◄── low cost, high throughput
                 │      :443           │
                 └────────┬────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌──────────┐┌──────────┐┌──────────┐
        │  OTel GW ││  OTel GW ││  OTel GW │  ◄── gateway mode, ASG
        │  :4317   ││  :4317   ││  :4317   │      buffer/batch/route
        └────┬─────┘└────┬─────┘└────┬─────┘
             │           │           │
             └─────┬─────┴─────┬─────┘
                   │           │
          ┌────────┴──┐  ┌─────┴──────┐  ┌──────────┐
          │Prometheus │  │    Loki    │  │  Tempo   │
          │   :9090   │  │   :3100    │  │:3200/4317│
          └───────────┘  └────────────┘  └──────────┘
                   ▲           ▲              ▲
                   │    datasource queries     │
                   └───────────┼───────────────┘
                               │
                          ┌────┴────┐
   Dashboard Users ──→    │   AWS    │  ◄── WAF → ALB (OIDC) :443
   HTTPS :443  WAF+ALB   │ Managed  │      (low traffic, L7, SSO)
                          │ Grafana  │
                          └──────────┘
```

### Why This Wins at 1000+ Endpoints

1. **Cost** — NLB is 60–70% cheaper than ALB for high-connection/high-throughput workloads; API Gateway is 5–10x more expensive.
2. **Performance** — NLB adds sub-millisecond latency; OTel gateway batches writes to backends, reducing per-request pressure on Prometheus/Loki/Tempo.
3. **Resilience** — Gateway collectors provide back-pressure, retry, and buffering; a Loki outage doesn't lose data, it queues.
4. **Simplicity for sources** — every OTel Collector pushes to one OTLP endpoint; the gateway handles fan-out so sources don't need per-backend exporters.
5. **Dashboard users still get ALB** — path routing, WAF, OIDC auth for the small number of human users where L7 features matter and cost is negligible.

---

## Estimated Monthly Cost — NLB + OTel Gateway + ALB

**Sizing basis:** 1000+ push endpoints · 10 dashboard users · 8 GB/day ingest · 90-day retention

| Component | Spec | Est. $/mo |
|-----------|------|-----------|
| NLB | TLS :443, ~1000 flows, ~240 GB/mo | $50–$120 |
| ALB | HTTPS :443, 10 users, minimal LCUs | $20–$30 |
| OTel Gateway (ASG) | 2–4× t3.medium | $60–$120 |
| AWS Managed Grafana (AMG) | AMG workspace (per-editor pricing) | $9/editor/mo; viewers free |
| Prometheus | 1× t3.large | $60 |
| Loki | 1× t3.large | $60 |
| Tempo | 1× t3.medium | $30 |
| EBS gp3 (hot / WAL) | ~180 GB (Prometheus TSDB, Loki WAL+index, Tempo WAL) | $14 |
| S3 (bulk store) | ~54 GB (Loki chunks + Tempo blocks at 90-day retention) | $1–$2 |
| ACM Certificates | 2 certs (NLB + ALB) | $0 |
| Route 53 | 1 hosted zone | $1 |
| Site-to-Site VPN | 1 tunnel (on-prem) | $36 |
| Data Transfer | ~240 GB in (free) + minimal out | $5–$10 |
| **Total (on-demand)** | | **$365–$515** |
| **Total (1-yr reserved)** | EC2 RI saves ~35% on compute | **$255–$360** |

> Inbound data transfer is free. EBS gp3 includes 3000 IOPS at no extra charge.  
> Compression ratios used: Prometheus ~3:1, Loki ~10:1, Tempo ~5:1.  
> S3 Standard pricing ~$0.023/GB/mo; at 54 GB compressed ≈ $1–$2/mo.

---

## Data Retention — 90-Day Policy

### Configuration per Backend

| Backend | Config Location | Setting | Default if Unset |
|---------|----------------|---------|------------------|
| **Prometheus** | CLI flag | `--storage.tsdb.retention.time=2160h` | 15 days |
| **Loki** | `loki.yaml` (two settings) | `limits_config.retention_period: 2160h` + `compactor.retention_enabled: true` | No deletion (infinite) |
| **Tempo** | `tempo.yaml` | `compactor.compaction.block_retention: 2160h` | No deletion (infinite) |

### How Retention is Enforced

- **Prometheus** — TSDB compaction automatically drops blocks older than `retention.time` and reclaims disk. No extra component needed.
- **Loki** — the **compactor** runs periodic retention sweeps, deleting chunks and index entries past `retention_period`. Both `retention_enabled: true` and the period must be set — omitting either keeps data forever.
- **Tempo** — the **compactor** deletes trace blocks older than `block_retention` during its compaction cycle. Without this flag, blocks accumulate indefinitely.

### EBS Sizing Note

Retention config alone does not prevent disk exhaustion. EBS volumes must hold the active working set (Prometheus TSDB for 90 days; Loki and Tempo WAL/index only — bulk data lives in S3). For Loki and Tempo, S3 handles the 90-day bulk retention automatically; EBS sizing for those two is driven by write-ahead log and index size, not total retention depth. Set CloudWatch alarms on EBS volume utilization (>80%) and monitor S3 bucket size to catch unexpected ingest growth before it causes issues.

> **Alert coverage:** disk saturation alerts are defined in [`alerting.md` — A-05](alerting.md#a-05-backend-storage-disk-saturation).

---

## Platform Self-Monitoring

The observability platform must be self-observable. An unmonitored platform fails silently — a dead Gateway, a full disk, or a saturated queue produces no errors visible to application teams, making it impossible to distinguish "no incidents" from "no telemetry."

The full alerting specification is maintained in [`alerting.md`](alerting.md). The table below summarises coverage and cross-references each alert definition.

| Component | Failure Mode | Alert | Mechanism |
|-----------|-------------|-------|-----------|
| OTel Gateway Collectors | Export failures — telemetry permanently dropped | [A-02](alerting.md#a-02-otel-gateway-export-errors) | Prometheus `otelcol_exporter_send_failed_*` → Grafana alert |
| OTel Gateway Collectors | Queue saturation — data loss imminent | [A-03](alerting.md#a-03-otel-gateway-queue-saturation) | Prometheus queue ratio → Grafana alert (warn 70%, critical 90%) |
| NLB (push ingress) | Unhealthy target hosts — inbound telemetry dropped at NLB | [A-04](alerting.md#a-04-nlb-target-group-unhealthy-hosts) | CloudWatch `HealthyHostCount` alarm → SNS |
| Prometheus / Loki / Tempo | Disk saturation — backend crash / data loss | [A-05](alerting.md#a-05-backend-storage-disk-saturation) | CW Agent `disk_used_percent` + `node_exporter` → CW alarm + Grafana alert |
| OTel Gateway ASG | Instance count below floor — degraded capacity | [A-06](alerting.md#a-06-otel-gateway-asg-instance-count-below-floor) | CloudWatch `GroupInServiceInstances` alarm → SNS |
| ECS task sidecars | Silent telemetry blackout from stopped sidecar | [A-01](alerting.md#a-01-ecs-sidecar-silent-telemetry-blackout) | EventBridge (ECS task state change) → SNS |

### OTel Platform Health Dashboard

A dedicated **OTel Platform Health** dashboard in AWS Managed Grafana provides a single-pane view of the platform's own health. Panel inventory:

- Gateway export error rate and enqueue failure rate (by exporter)
- Gateway queue fill % (warn threshold line at 70%, critical at 90%)
- Gateway spans / metric points / log records received and exported per second (by receiver and exporter)
- Gateway CPU utilization per ASG instance
- NLB healthy host count and processed bytes
- Prometheus, Loki, Tempo disk utilization % (with alert threshold markers)
- ASG in-service instance count
- ECS sidecar fatal exit count (rolling 1 h)

All alert rules link directly to panels on this dashboard so operators see the full platform health picture during an incident.

### Alerting Channels

| Topic (SNS) | Severity | Subscribers |
|-------------|----------|-------------|
| `observability_alerts_critical` | Critical | PagerDuty (P1 page), Slack `#observability-alerts`, on-call email DL |
| `observability_alerts_warning` | Warning | Slack `#observability-alerts`, on-call email DL |

---

## Key Design Decisions

1. **Split NLB (push) + ALB (dashboards)** — right-sizes cost and features for each traffic profile.
2. **OTel Collector gateway tier** — buffers, batches, routes, and provides back-pressure between 1000+ sources and backends.
3. **All external push over HTTPS :443** — NLB TLS listener; OTel Collectors push OTLP to a single endpoint.
4. **Private subnets for backends** — Prometheus, Loki, Tempo are never directly internet-reachable.
5. **AWS Managed Grafana (AMG) with WAF → ALB (OIDC)** — no EC2 to manage for the dashboard tier; WAF filters inbound requests, ALB enforces OIDC before any request reaches the AMG workspace.
6. **Hybrid storage (EBS gp3 + S3)** — EBS gp3 for hot working sets (Prometheus TSDB, Loki WAL + index, Tempo WAL); S3 for bulk chunk and block storage on Loki and Tempo. Reduces storage cost ~60% vs all-EBS (~$15/mo vs ~$40/mo), with S3's multi-AZ durability as a side benefit. Prometheus uses EBS only — its TSDB is local-disk by design and does not support object storage.
7. **90-day retention** — explicitly configured per backend; Loki supports per-tenant overrides for differentiated policies.
