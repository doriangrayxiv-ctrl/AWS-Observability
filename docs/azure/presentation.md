# Presentation Layer — Grafana Platform (Azure)

## Overview

Azure Managed Grafana serves as the single pane of glass for dashboards, alerting, and exploration. Dashboard users access Grafana at `grafana.company.com:443` through an Azure Front Door (WAF) → Managed Grafana (Entra ID SSO) access path; only authenticated sessions reach the Grafana workspace, where Grafana RBAC then controls what each user can see and do. Backend datastores (Prometheus, Loki, Tempo) run on Azure VMs in private subnets and receive telemetry pushed from authenticated sources across multiple Azure regions, on-prem environments, and DigitalOcean.

> **Key difference from AWS design:** Azure Managed Grafana integrates Microsoft Entra ID natively for SSO — authentication is enforced at the Grafana workspace itself, not at an intermediate load balancer. Azure Front Door provides WAF protection (rate-limiting, geo/IP filtering) in front of Grafana, but OIDC session enforcement is built into the Grafana service. This simplifies the query path compared to the ALB+OIDC pattern in AWS.

---

## Azure Resources

| Resource | Purpose | Notes |
|----------|---------|-------|
| **Azure Load Balancer Standard** | L4 TCP pass-through for high-volume telemetry push traffic | Fronts OTel Gateway VMSS; 1000+ endpoint scale; no TLS termination — end-to-end encryption to Gateway |
| **Azure Front Door Standard + WAF** | Inspects and filters inbound HTTPS requests to Grafana; managed TLS certificates | Rate-limiting, geo/IP rules, bot protection; WAF policy attached |
| **Azure Managed Grafana** (Standard tier) | Managed Grafana workspace; Entra ID SSO built-in; enforces Grafana RBAC post-authentication | `grafana.company.com:443`; Entra ID groups mapped to Viewer / Editor / Admin roles |
| **OTel Collector Gateway (VMSS)** | Receive, buffer, batch, route telemetry to backends | 2–4 instances (B2ms), Virtual Machine Scale Set |
| **Azure Key Vault** | TLS certificates for Gateway instances; secrets for external integrations (Snowflake, POS, Shopify) | Certificate auto-rotation; RBAC-scoped access |
| **Azure VMs** | Run Prometheus, Loki, Tempo | Sized per retention/volume needs; no VM for Grafana |
| **Azure Managed Disks (Premium SSD v2)** | Hot storage: Prometheus TSDB, Loki WAL + index, Tempo WAL | ~180 GB total; sized for active working set only |
| **Azure Blob Storage (Hot tier)** | Bulk/cold storage: Loki chunks, Tempo trace blocks | ~54 GB at 90-day retention; LRS or ZRS durability; Prometheus does not use Blob Storage |
| **Network Security Groups (NSG)** | Control inbound/outbound traffic per service | See port table below |
| **Azure Virtual Network (VNet)** | Isolated network for the observability stack | Private subnets for backends, public subnet for LB |
| **VNet Peering** | Cross-region connectivity from other Azure regions | Peering per region |
| **Azure VPN Gateway** | On-prem → VNet and DigitalOcean → VNet connectivity | S2S IPsec or WireGuard tunnel |
| **Azure DNS** | DNS for `grafana.company.com` / `gateway.company.com` | CNAME to Front Door endpoint; A record to LB public IP |
| **Azure Managed Identity** | Service-level permissions, Key Vault access | System-assigned per VM/VMSS; least-privilege per component |

---

## Ports & Protocols

Two separate traffic profiles flow through this architecture: high-volume machine push (Azure LB path) and low-volume human dashboard access (Front Door path). The table is grouped by flow type for clarity.

### External Entry Points

| Port | Protocol | Direction | Listener | Caller (source) |
|------|----------|-----------|----------|-----------------|
| **443** | TCP (TLS pass-through) | Inbound | Azure LB Standard → OTel Gateway VMSS | OTel Collectors (all sources — Azure, on-prem, DigitalOcean); OTLP push |
| **443** | HTTPS | Inbound | Azure Front Door (WAF) → Managed Grafana | Dashboard users (browser); Entra ID SSO enforced at Grafana |

### Internal — OTel Gateway Receivers (Azure LB forwards inbound :443 collector traffic here)

| Port | Protocol | Direction | Listener | Caller (source) |
|------|----------|-----------|----------|-----------------|
| **4317** | gRPC (TLS) | Internal | OTel Gateway (OTLP gRPC receiver with TLS) | Azure LB; forwards all inbound collector OTLP gRPC traffic |
| **4318** | HTTP (TLS) | Internal | OTel Gateway (OTLP HTTP receiver with TLS) | Azure LB; forwards all inbound collector OTLP HTTP traffic |

> **TLS at the Gateway:** Unlike AWS NLB which can terminate TLS using ACM certificates, Azure Load Balancer Standard is pure L4 and passes TCP streams without TLS termination. The OTel Gateway instances terminate TLS themselves using certificates sourced from Azure Key Vault. This provides true end-to-end encryption from collector to Gateway — no unencrypted hop between the load balancer and the backend.

### Internal — OTel Gateway → Backends (Gateway is the sole writer; no source writes directly to a backend)

| Port | Protocol | Direction | Listener | Caller (source) |
|------|----------|-----------|----------|-----------------|
| **9090/api/v1/write** | HTTP | Internal | Prometheus (remote-write ingest) | OTel Gateway (Prometheus remote-write exporter; metrics only) |
| **3100/loki/api/v1/push** | HTTP | Internal | Loki (log push ingest) | OTel Gateway (Loki exporter; logs only) |
| **4317** | gRPC | Internal | Tempo (OTLP trace ingest) | OTel Gateway (OTLP exporter; traces only) |

### Internal — Managed Grafana → Backends (read-only datasource queries)

| Port | Protocol | Direction | Listener | Caller (source) |
|------|----------|-----------|----------|-----------------|
| **9090** | HTTP | Internal | Prometheus (query API) | Azure Managed Grafana (datasource queries via managed private endpoint) |
| **3100** | HTTP | Internal | Loki (query API) | Azure Managed Grafana (datasource queries via managed private endpoint) |
| **3200** | HTTP | Internal | Tempo (query API) | Azure Managed Grafana (datasource queries via managed private endpoint) |

> **All internal traffic** stays within the VNet on private subnets. All backend ports are unreachable from outside the VNet.
> **Azure LB Standard is the sole external entry point for telemetry.** It forwards to the OTel Gateway VMSS, which is the sole writer to all backends (Prometheus, Loki, Tempo). No collector or external source writes to a backend directly.
> **Azure Front Door is the sole external entry point for dashboard users.** It carries read-only datasource queries only and is never in the telemetry push path.
> **Managed Grafana → Backend connectivity** uses managed private endpoints to reach Prometheus, Loki, and Tempo on private IPs within the VNet.

---

## Authentication & Authorization

| Flow | Method |
|------|--------|
| Dashboard users → Grafana | HTTPS 443 — Azure Front Door (WAF) → Azure Managed Grafana with Entra ID SSO; authentication handled natively by the Grafana workspace, not at a load balancer |
| OTel Collectors → Gateway LB | TCP 443 — mTLS at the OTel Gateway TLS listener, or `bearertokenauth` extension in OTel Collector config; Azure LB is not in the auth path (L4 pass-through) |

### Grafana RBAC (post-authentication authorization)

Once a user's identity is confirmed by Entra ID at the Managed Grafana workspace, Azure RBAC role assignments and Grafana's built-in RBAC control what they can see and do:

| Control | Behaviour |
|---------|-----------|
| **Azure RBAC role assignments** | Entra ID groups are mapped to Azure Managed Grafana roles: **Grafana Viewer**, **Grafana Editor**, **Grafana Admin** via Azure role assignments on the Grafana resource |
| **Folder-level permissions** | Restrict which teams can access which dashboards (e.g. Ops team → infra dashboards; Dev team → application dashboards) |
| **Data source permissions** | Control which roles can query Prometheus, Loki, or Tempo via Explore — limited to Editor+ by default |
| **Team-based assignments** | Users are assigned to teams aligned to Entra ID groups; permissions are granted at the team level for bulk management |

This creates a two-step access model: **Entra ID enforces authentication** (no unauthenticated request reaches Grafana), and **Grafana RBAC enforces authorization** (authenticated users see only what their role permits).

> **Azure RBAC advantage:** Unlike AWS IAM Identity Center which requires mapping OIDC claims to Grafana roles at the workspace level, Azure Managed Grafana uses native Azure RBAC role assignments. Entra ID group → Azure role assignment → Grafana role is managed through standard Azure IAM tooling (Portal, Terraform `azurerm_role_assignment`, Azure CLI). No separate SAML/OIDC claim mapping configuration is required.

---

## ASCII Architecture Map

```
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │                         EXTERNAL SOURCES                                    │
 │                                                                              │
 │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
 │   │Azure Region A│  │Azure Region B│  │  On-Prem DC  │  │ DigitalOcean │   │
 │   │  OTel Coll.  │  │  OTel Coll.  │  │  OTel Coll.  │  │  OTel Coll.  │   │
 │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
 │          │                  │                  │                  │           │
 │          │  OTLP :443       │  OTLP :443       │  VPN + :443     │  VPN+:443 │
 └──────────┼──────────────────┼──────────────────┼──────────────────┼───────────┘
            │                  │                  │                  │
            ▼                  ▼                  ▼                  ▼
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │                      Azure — Observability VNet                             │
 │                                                                              │
 │  PUSH PATH (high volume — 1000+ sources)                                    │
 │   ┌──────────────────────────────────────┐                                  │
 │   │   Azure LB Standard  (TCP :443)      │  ◄── L4 pass-through, cheap     │
 │   │   Static Public IP · No TLS term.    │      end-to-end TLS to Gateway  │
 │   └──────────┬──────────┬────────────────┘                                  │
 │              │          │          │                                         │
 │              ▼          ▼          ▼                                         │
 │        ┌──────────┐┌──────────┐┌──────────┐                                 │
 │        │  OTel GW ││  OTel GW ││  OTel GW │  ◄── VMSS, gateway mode        │
 │        │ :4317 TLS││ :4317 TLS││ :4317 TLS│     TLS term + buffer/batch    │
 │        └────┬─────┘└────┬─────┘└────┬─────┘                                 │
 │             └─────┬─────┴─────┬─────┘                                       │
 │                   │           │                                              │
 │          ┌────────┴──┐  ┌─────┴──────┐  ┌──────────┐                        │
 │          │Prometheus │  │    Loki    │  │  Tempo   │   Private Subnets      │
 │          │   :9090   │  │   :3100    │  │:3200/4317│                        │
 │          └─────┬─────┘  └─────┬──────┘  └────┬─────┘                        │
 │                │              │               │                              │
 │                │ managed private endpoints     │                              │
 │                └──────┬───────┴───────────────┘                              │
 │                       │                                                      │
 │  QUERY PATH (low volume — dashboard users)                                  │
 │   ┌──────────────────────────────────────┐                                  │
 │   │  Azure Front Door  (HTTPS :443)      │  ◄── WAF, managed TLS, CDN      │
 │   │  WAF Policy · Rate-limiting · Geo    │                                  │
 │   └──────────────────┬───────────────────┘                                  │
 │                      │                                                       │
 │               ┌──────────────────────┐                                      │
 │               │  Azure Managed       │                                      │
 │               │  Grafana             │                                      │
 │               │  (Entra ID SSO)      │                                      │
 │               └──────────────────────┘                                      │
 │                                                                              │
 │   ┌──────────────────────────────────────────────────────────────┐          │
 │   │  Persistent Storage (90-day retention)                       │          │
 │   │                                                              │          │
 │   │  Managed Disks (hot / WAL)       Blob Storage (bulk, 90-day) │          │
 │   │  Prometheus TSDB  ~150 GB        Loki chunks    ~36 GB      │          │
 │   │  Loki WAL + index  ~20 GB        Tempo blocks   ~18 GB      │          │
 │   │  Tempo WAL         ~10 GB        (Prometheus: disk only)    │          │
 │   └──────────────────────────────────────────────────────────────┘          │
 │                                                                              │
 │   Dashboard Users ─── HTTPS :443 ──→ Front Door (WAF) → Managed Grafana   │
 │   1000+ Collectors ── OTLP :443 ──→ Azure LB → OTel GW (TLS) → backends   │
 └──────────────────────────────────────────────────────────────────────────────┘
```

---

## Ingress Strategy — Choosing the Right Load Balancer for 1000+ Push Sources

With 1000+ OTel Collectors continuously pushing metrics, logs, and traces, the ingress layer is the highest-traffic component in the stack. The table below compares viable Azure approaches.

### Option Comparison

#### Option A: Azure Application Gateway (with WAF v2)

| Aspect | Detail |
|--------|--------|
| **How it works** | Layer 7 LB; terminates TLS, inspects HTTP paths, routes to backend pools |
| **Path routing** | Native — route by URL path to different backend pools |
| **gRPC support** | Yes (HTTP/2 backend setting) |
| **Auth** | Header inspection, Entra ID integration via OIDC |
| **Scaling** | Autoscaling v2 SKU; capacity units determine throughput |

| Pros | Cons |
|------|------|
| Rich L7 features (path routing, WAF, OIDC) | **Expensive** — fixed cost ~$146/mo (gateway) + ~$117/mo (WAF) = **$263/mo base** before capacity units |
| Native managed TLS termination | Capacity units add ~$5.80/CU/mo; at sustained 1000+ connections expect 10–20+ CUs |
| Built-in WAF v2 | Higher latency than L4 — full HTTP parse on every request |
| Good for low-volume dashboard access | Massive overkill for telemetry push where all traffic goes to the same backend tier |

**Estimated cost at scale:** $300–$500/mo (gateway + WAF + CUs)

---

#### Option B: Azure Load Balancer Standard — Recommended for Push Traffic

| Aspect | Detail |
|--------|--------|
| **How it works** | Layer 4 LB; passes TCP streams directly to backend instances, no HTTP inspection |
| **Path routing** | None — routes by port/protocol only |
| **gRPC support** | Yes (TCP pass-through; TLS terminated at backend) |
| **Auth** | No header inspection — auth must happen at the application layer (OTel Gateway) |
| **Scaling** | Handles millions of connections; designed for high-throughput, low-latency |

| Pros | Cons |
|------|------|
| **Dramatically cheaper** — ~$18/mo fixed + minimal data processing charges | No path-based routing — need a gateway tier behind it |
| Sub-millisecond latency (no L7 parsing) | No WAF attachment |
| Static Public IP available — easier for on-prem firewall rules | No TLS termination — TLS must terminate at backend (but this is end-to-end encryption, arguably better) |
| Handles millions of concurrent flows natively | Auth pushed to OTel Gateway layer |
| No capacity-unit pricing — flat fee regardless of throughput | Less visibility than L7 logs |

**Estimated cost at scale:** $19–$25/mo

---

#### Option C: Azure LB Standard + OTel Collector Gateway VMSS — Best Fit ✓

| Aspect | Detail |
|--------|--------|
| **How it works** | Azure LB Standard fronts a horizontally-scaled fleet of OTel Collectors in **gateway mode** on a VMSS; gateway collectors terminate TLS, receive OTLP on :4317/:4318, then route to Prometheus, Loki, Tempo |
| **Path routing** | Not needed — the OTel Gateway pipeline handles routing via exporters |
| **gRPC support** | Native OTLP gRPC at the collector (TLS terminated by the collector, not the LB) |
| **Auth** | `bearertokenauth` extension or mTLS at OTel Gateway TLS listener |
| **Scaling** | Azure LB + VMSS auto-scale rules; scale on CPU/network |

| Pros | Cons |
|------|------|
| **Cheapest at scale** — Azure LB ~$20/mo + VMSS instances for gateway | Extra compute layer (gateway fleet), but these are lightweight |
| **End-to-end TLS** — no unencrypted hop between LB and backend (Azure LB passes TCP, Gateway terminates TLS) | More moving parts to operate |
| **Buffering & back-pressure** — gateway collectors queue, batch, retry; protects backends from burst spikes | Requires OTel Collector config management for gateway tier |
| **Unified OTLP ingress** — all sources push to one OTLP endpoint; gateway routes metrics→Prometheus, logs→Loki, traces→Tempo | Must monitor gateway health (but OTel Collector exports its own metrics) |
| **Protocol normalization** — accepts OTLP gRPC, OTLP HTTP all on one endpoint | |
| **Decouples sources from backends** — backend migration/scaling is invisible to push sources | |

**Estimated cost at scale:** Azure LB ~$20/mo + 2–4 gateway instances (B2ms ~$60/ea) = **$140–$260/mo total**

---

#### Alternatives Considered

- **Option D — Azure API Management:** Rejected. At 1000+ endpoints pushing every 15 seconds, request volume reaches ~170M/month. API Management consumption tier pricing (~$3.50/M calls) costs ~$600/mo, and the premium tier is vastly more expensive. Gateway latency and payload limits are not suited to streaming OTLP/gRPC telemetry.
- **Option E — Self-Managed Reverse Proxy (Nginx/Envoy on VM):** Rejected. Operationally redundant — duplicates the routing and TLS termination the OTel Collector Gateway already handles, adding a separate HA/scaling/patching burden without the buffering and processing benefits the gateway tier provides.

---

### Recommendation: Split Architecture (Azure LB + OTel Gateway + Front Door)

Separate the two traffic profiles since they have fundamentally different characteristics:

| Traffic Profile | Volume | Azure Service | Target |
|----------------|--------|---------------|--------|
| **Dashboard users** (low-volume, human) | Low — tens of concurrent users | **Azure Front Door Standard** (WAF) on :443 | Azure Managed Grafana |
| **Telemetry push** (high-volume, machine) | High — 1000+ persistent pushers | **Azure LB Standard** on :443 | OTel Gateway VMSS :4317/:4318 |

```
                    1000+ OTel Collectors
                    (Azure, on-prem, DO)
                            │
                     OTLP HTTPS :443
                            │
                            ▼
                 ┌─────────────────────┐
                 │  Azure LB Standard  │  ◄── L4 TCP, flat fee, ~$20/mo
                 │    (TCP :443)       │
                 └────────┬────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌──────────┐┌──────────┐┌──────────┐
        │  OTel GW ││  OTel GW ││  OTel GW │  ◄── gateway mode, VMSS
        │ :4317 TLS││ :4317 TLS││ :4317 TLS│     TLS term/buffer/batch
        └────┬─────┘└────┬─────┘└────┬─────┘
             │           │           │
             └─────┬─────┴─────┬─────┘
                   │           │
          ┌────────┴──┐  ┌─────┴──────┐  ┌──────────┐
          │Prometheus │  │    Loki    │  │  Tempo   │
          │   :9090   │  │   :3100    │  │:3200/4317│
          └───────────┘  └────────────┘  └──────────┘
                   ▲           ▲              ▲
                   │  managed private endpoints│
                   └───────────┼───────────────┘
                               │
                          ┌────┴────┐
   Dashboard Users ──→    │  Azure   │  ◄── Front Door (WAF) :443
   HTTPS :443  WAF        │ Managed  │      Entra ID SSO (built-in)
                          │ Grafana  │
                          └──────────┘
```

### Why This Wins at 1000+ Endpoints

1. **Cost** — Azure LB Standard is dramatically cheaper than Application Gateway ($20/mo vs $300+/mo); API Management is 5–10x more expensive.
2. **End-to-end TLS** — Azure LB passes TCP without terminating TLS; OTel Gateway terminates TLS using Key Vault certificates. No unencrypted hop between LB and backend.
3. **Performance** — Azure LB adds sub-millisecond latency; OTel Gateway batches writes to backends, reducing per-request pressure on Prometheus/Loki/Tempo.
4. **Resilience** — Gateway collectors provide back-pressure, retry, and buffering; a Loki outage doesn't lose data, it queues.
5. **Simplicity for sources** — every OTel Collector pushes to one OTLP endpoint; the gateway handles fan-out so sources don't need per-backend exporters.
6. **Dashboard users still get WAF** — Azure Front Door provides WAF, rate-limiting, and managed TLS certificates for the small number of human users where protection matters and cost is negligible.

---

## Estimated Monthly Cost — Azure LB + OTel Gateway + Front Door

**Sizing basis:** 1000+ push endpoints · 10 dashboard users · 8 GB/day ingest · 90-day retention

| Component | Spec | Est. $/mo |
|-----------|------|-----------|
| Azure LB Standard | TCP :443, ~1000 flows, ~240 GB/mo processed | $20–$25 |
| Azure Front Door Standard | HTTPS :443, WAF policy, 10 users | $40–$45 |
| OTel Gateway (VMSS) | 2–4× B2ms (2 vCPU, 8 GB) | $120–$240 |
| Azure Managed Grafana | Standard tier workspace (no per-editor charge) | $73 |
| Prometheus VM | 1× B2ms (2 vCPU, 8 GB) | $60 |
| Loki VM | 1× B2ms (2 vCPU, 8 GB) | $60 |
| Tempo VM | 1× B2s (2 vCPU, 4 GB) | $30 |
| Managed Disks (Premium SSD v2) | ~180 GB (Prometheus TSDB, Loki WAL+index, Tempo WAL) | $17 |
| Azure Blob Storage (Hot) | ~54 GB (Loki chunks + Tempo blocks at 90-day) | $1–$2 |
| Azure Key Vault | Certificates + secrets; key operations | $5 |
| Azure DNS | 1 DNS zone + records | $1 |
| Azure VPN Gateway (VpnGw1) | 1× gateway + S2S connections (on-prem + DO) | $140 |
| Data Transfer | ~240 GB in (free) + minimal egress | $5–$10 |
| **Total (pay-as-you-go)** | | **$572–$748** |
| **Total (1-yr reserved)** | VM Reserved Instances save ~35% on compute | **$410–$540** |

> **Cost comparison notes:**
> - Azure Load Balancer Standard is dramatically cheaper than AWS NLB for this workload ($20 vs $50–$120) due to flat-fee pricing with no per-NLCU charges.
> - Azure Managed Grafana Standard tier uses per-workspace pricing (~$73/mo) versus AWS AMG's per-editor pricing ($9/editor/mo × 10 = $90/mo). At 10 editors the costs are comparable; Azure is cheaper at <8 editors, AWS is cheaper at >8 editors.
> - Azure VPN Gateway (VpnGw1 SKU at ~$140/mo) is more expensive than AWS Site-to-Site VPN (~$72/mo for 2 connections). This is the largest cost delta. The Basic SKU ($27/mo) supports S2S but lacks BGP and zone redundancy — acceptable for non-production; production should use VpnGw1.
> - Azure Front Door Standard + WAF ($40–$45/mo) is comparable to AWS WAF + ALB ($28–$40/mo).
> - Inbound data transfer is free. Premium SSD v2 includes configurable IOPS at no extra charge for the base provisioned capacity.
> - Compression ratios used: Prometheus ~3:1, Loki ~10:1, Tempo ~5:1.
> - Blob Storage Hot tier ~$0.018/GB/mo; at 54 GB compressed ≈ $1–$2/mo.

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

### Managed Disk Sizing Note

Retention config alone does not prevent disk exhaustion. Managed Disks must hold the active working set (Prometheus TSDB for 90 days; Loki and Tempo WAL/index only — bulk data lives in Blob Storage). For Loki and Tempo, Blob Storage handles the 90-day bulk retention automatically; disk sizing for those two is driven by write-ahead log and index size, not total retention depth. Set Azure Monitor alerts on disk utilisation (>80%) and monitor Blob Storage container size to catch unexpected ingest growth before it causes issues.

> **Alert coverage:** disk saturation alerts are defined in [alerting.md — A-05](alerting.md#a-05-backend-storage-disk-saturation).

---

## Platform Self-Monitoring

The observability platform must be self-observable. An unmonitored platform fails silently — a dead Gateway, a full disk, or a saturated queue produces no errors visible to application teams, making it impossible to distinguish "no incidents" from "no telemetry."

The full alerting specification is maintained in [alerting.md](alerting.md). The table below summarises coverage and cross-references each alert definition.

| Component | Failure Mode | Alert | Mechanism |
|-----------|-------------|-------|-----------|
| OTel Gateway Collectors | Export failures — telemetry permanently dropped | [A-02](alerting.md#a-02-otel-gateway-export-errors) | Prometheus `otelcol_exporter_send_failed_*` → Grafana alert |
| OTel Gateway Collectors | Queue saturation — data loss imminent | [A-03](alerting.md#a-03-otel-gateway-queue-saturation) | Prometheus queue ratio → Grafana alert (warn 70%, critical 90%) |
| Azure LB (push ingress) | Unhealthy backend pool — inbound telemetry dropped at LB | [A-04](alerting.md#a-04-azure-lb-backend-pool-unhealthy) | Azure Monitor `HealthProbeStatus` → Action Group |
| Prometheus / Loki / Tempo | Disk saturation — backend crash / data loss | [A-05](alerting.md#a-05-backend-storage-disk-saturation) | Azure Monitor Agent `disk_used_percent` + `node_exporter` → Azure Monitor alert + Grafana alert |
| OTel Gateway VMSS | Instance count below floor — degraded capacity | [A-06](alerting.md#a-06-otel-gateway-vmss-instance-count-below-floor) | Azure Monitor `VMSS instance count` → Action Group |
| ACA sidecar containers | Sidecar restart / crash affecting telemetry | [A-01](alerting.md#a-01-aca-sidecar-telemetry-disruption) | Azure Monitor container diagnostics → Action Group |

### OTel Platform Health Dashboard

A dedicated **OTel Platform Health** dashboard in Azure Managed Grafana provides a single-pane view of the platform's own health. Panel inventory:

- Gateway export error rate and enqueue failure rate (by exporter)
- Gateway queue fill % (warn threshold line at 70%, critical at 90%)
- Gateway spans / metric points / log records received and exported per second (by receiver and exporter)
- Gateway CPU utilization per VMSS instance
- Azure LB backend pool health status and data path availability
- Prometheus, Loki, Tempo disk utilization % (with alert threshold markers)
- VMSS instance count (current vs minimum)
- ACA sidecar restart count (rolling 1h)

All alert rules link directly to panels on this dashboard so operators see the full platform health picture during an incident.

### Alerting Channels

| Action Group | Severity | Receivers |
|-------------|----------|-----------|
| `observability-alerts-critical` | Critical | PagerDuty (webhook), Slack `#observability-alerts` (webhook), on-call email DL |
| `observability-alerts-warning` | Warning | Slack `#observability-alerts` (webhook), on-call email DL |

---

## Key Design Decisions

1. **Split Azure LB (push) + Front Door (dashboards)** — right-sizes cost and features for each traffic profile.
2. **OTel Collector Gateway VMSS** — buffers, batches, routes, and provides back-pressure between 1000+ sources and backends.
3. **End-to-end TLS (no LB-level termination)** — Azure LB Standard passes TCP; OTel Gateway terminates TLS using Key Vault certs. More secure than TLS termination at the LB (no unencrypted hop).
4. **Private subnets for backends** — Prometheus, Loki, Tempo are never directly internet-reachable.
5. **Azure Managed Grafana with Entra ID SSO** — no VM to manage for the dashboard tier; Entra ID SSO is built-in (no OIDC configuration at a load balancer); Azure RBAC maps Entra groups to Grafana roles natively.
6. **Azure Front Door for WAF** — provides rate-limiting, geo/IP filtering, and managed TLS certificates at lower cost than Application Gateway WAF v2 for this low-traffic use case.
7. **Hybrid storage (Managed Disks + Blob Storage)** — Premium SSD v2 for hot working sets (Prometheus TSDB, Loki WAL + index, Tempo WAL); Blob Storage Hot tier for bulk chunk and block storage on Loki and Tempo. Reduces storage cost ~60% vs all-disk, with Blob Storage LRS/ZRS durability as a side benefit. Prometheus uses disk only — its TSDB is local-disk by design and does not support object storage.
8. **90-day retention** — explicitly configured per backend; Loki supports per-tenant overrides for differentiated policies.
9. **Azure DevOps Pipelines + Terraform + Ansible** — all infrastructure provisioned via Terraform; VM-level configuration managed by Ansible; pipelines run from Azure DevOps with source in Azure Git Repos.
