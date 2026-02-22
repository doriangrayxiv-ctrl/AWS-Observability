# Presentation Layer — Grafana Platform

## Overview

Self-hosted (unmanaged) Grafana running in AWS, serving as the single pane of glass for dashboards, alerting, and exploration. Backend datastores (Prometheus, Loki, Tempo) are co-located and receive telemetry pushed from authenticated sources across multiple AWS regions and on-prem environments.

---

## AWS Resources

| Resource | Purpose | Notes |
|----------|---------|-------|
| **NLB** (Network Load Balancer) | TLS termination for high-volume telemetry push traffic | Fronts OTel Gateway fleet; 1000+ endpoint scale |
| **ALB** (Application Load Balancer) | HTTPS termination for Grafana dashboard users | Low-traffic L7 with path routing, WAF, OIDC |
| **OTel Collector Gateway (ASG)** | Receive, buffer, batch, route telemetry to backends | 2–4 instances (t3.medium), Auto Scaling Group |
| **ACM Certificate** | TLS certs for NLB + ALB HTTPS listeners | Auto-renewing via AWS Certificate Manager |
| **EC2 Instances** (or ECS tasks) | Run Grafana, Prometheus, Loki, Tempo | Sized per retention/volume needs |
| **EBS Volumes** | Persistent storage for Prometheus TSDB, Loki chunks, Tempo blocks | gp3; sized for 90-day retention |
| **Security Groups** | Control inbound/outbound traffic per service | See port table below |
| **VPC** | Isolated network for the observability stack | Private subnets for backends, public subnet for ALB |
| **VPC Peering / TGW** | Cross-region connectivity from other AWS regions | Peering or Transit Gateway per region |
| **Site-to-Site VPN** | On-prem → VPC connectivity | IPsec or WireGuard tunnel |
| **Route 53** | DNS for `grafana.company.internal` / public FQDN | Alias to ALB |
| **IAM Roles** | Service-level permissions, ALB access logs | Least-privilege per component |

---

## Ports & Protocols

| Port | Protocol | Direction | Service | Consumer |
|------|----------|-----------|---------|----------|
| **443** | HTTPS | Inbound | ALB → Grafana | Dashboard users (browser) |
| **443** | HTTPS | Inbound | ALB → push endpoints | OTel Collectors (remote-write, OTLP) |
| **3000** | HTTP | Internal | Grafana | ALB target group |
| **9090** | HTTP | Internal | Prometheus (query) | Grafana datasource |
| **9090/api/v1/write** | HTTP | Internal | Prometheus (remote-write) | ALB target group (push) |
| **3100** | HTTP | Internal | Loki (query + push) | Grafana datasource / ALB target group |
| **3200** | HTTP | Internal | Tempo (query) | Grafana datasource |
| **4317** | gRPC | Internal | Tempo OTLP (traces push) | ALB target group (push) |
| **4318** | HTTP | Internal | Tempo OTLP (traces push) | ALB target group (push) |

> **All internal traffic** stays within the VPC on private subnets.  
> **All external push traffic** terminates at the ALB on port 443 and is routed to the appropriate backend target group by path/host rules.

---

## Authentication

| Flow | Method |
|------|--------|
| Dashboard users → Grafana | HTTPS 443 — Grafana built-in auth (local, LDAP, or OAuth/SSO) |
| OTel Collectors → push endpoints | HTTPS 443 — Bearer token or mTLS via ALB; per-source API keys |

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
 │                 ┌────┴────┐                                                  │
 │                 │ Grafana │                                                  │
 │                 │  :3000  │                                                  │
 │                 └─────────┘                                                  │
 │                                                                              │
 │   ┌──────────────────────────────┐                                          │
 │   │        EBS (gp3)             │                                          │
 │   │  Prometheus TSDB │ Loki WAL  │                                          │
 │   │  Loki Chunks │ Tempo Blocks  │                                          │
 │   │       90-day retention       │                                          │
 │   └──────────────────────────────┘                                          │
 │                                                                              │
 │   Dashboard Users ─── HTTPS :443 ──→ ALB ──→ Grafana :3000                 │
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

#### Option D: API Gateway (REST)

| Pros | Cons |
|------|------|
| Fully managed, auto-scales infinitely | **Extremely expensive at volume** — $3.50/million requests; 1000 endpoints pushing every 15s ≈ 170M req/mo = **~$600/mo in API calls alone** |
| Built-in auth (API keys, IAM, Cognito) | Not designed for streaming/gRPC telemetry workloads |
| Usage plans and throttling | 30-second timeout; 10MB payload limit |
| | Adds latency (full request/response cycle per call) |

**Verdict:** Too expensive and wrong tool for high-throughput telemetry push.

---

#### Option E: Self-Managed Reverse Proxy (Nginx / Envoy on EC2)

| Pros | Cons |
|------|------|
| Full control over routing, TLS, auth | **You own the uptime** — must build HA, scaling, patching |
| No per-request AWS charges | Need to pair with NLB or EIP for a stable entry point anyway |
| Envoy has native gRPC, L7 routing, observability | Operational burden; duplicates what OTel gateway already does |
| Can be very cheap (EC2 cost only) | No auto-scaling unless you build it |

**Verdict:** Viable but unnecessary when OTel Collector gateway mode provides the same routing + adds buffering/processing.

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
   Dashboard Users ──→    │ Grafana │  ◄── fronted by ALB :443
        HTTPS :443   ALB  │  :3000  │      (low traffic, L7 features)
                          └─────────┘
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
| Grafana | 1× t3.medium | $30 |
| Prometheus | 1× t3.large | $60 |
| Loki | 1× t3.large | $60 |
| Tempo | 1× t3.medium | $30 |
| EBS gp3 (all backends) | ~470 GB total | $40 |
| ACM Certificates | 2 certs (NLB + ALB) | $0 |
| Route 53 | 1 hosted zone | $1 |
| Site-to-Site VPN | 1 tunnel (on-prem) | $36 |
| Data Transfer | ~240 GB in (free) + minimal out | $5–$10 |
| **Total (on-demand)** | | **$390–$540** |
| **Total (1-yr reserved)** | EC2 RI saves ~35% on compute | **$280–$380** |

> Inbound data transfer is free. EBS gp3 includes 3000 IOPS at no extra charge.  
> Compression ratios used: Prometheus ~3:1, Loki ~10:1, Tempo ~5:1.

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

Retention config alone does not prevent disk exhaustion. EBS volumes must hold 90 days at full ingest rate. Set CloudWatch alarms on volume utilization (>80%) to expand storage before housekeeping falls behind ingestion.

---

## Key Design Decisions

1. **Split NLB (push) + ALB (dashboards)** — right-sizes cost and features for each traffic profile.
2. **OTel Collector gateway tier** — buffers, batches, routes, and provides back-pressure between 1000+ sources and backends.
3. **All external push over HTTPS :443** — NLB TLS listener; OTel Collectors push OTLP to a single endpoint.
4. **Private subnets for backends** — Prometheus, Loki, Tempo are never directly internet-reachable.
5. **EBS gp3** — cost-effective, tunable IOPS storage for 90-day retention volumes.
6. **90-day retention** — explicitly configured per backend; Loki supports per-tenant overrides for differentiated policies.
