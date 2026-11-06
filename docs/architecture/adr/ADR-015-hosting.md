# ADR-015: Hosting: managed EU serving plane (Scaleway) + Hetzner batch plane

- Status: **Accepted** (owner, 2026-11-06, Q-02b: option A). Implementation details validated by spike S-1 in M3; option C stays the fallback.
- Date: 2026-11-06 (v0.2 replaces v0.1, which put everything on a single VPS)
- Deciders: architect (design mode); owner for cost

## Context
- NFR-10 (v0.2): €50–200/month for hosting, data, prices and email, **excluding LLM tokens**. The owner **prefers managed cloud services** if the budget allows. Otherwise a mix of managed services and cheaper Hetzner servers for heavy batch work such as the historic download.
- NFR-08: EU hosting for personal data. NFR-06: 99 % availability. NFR-01: ≤100-user beta (confirmed), design for ~10k. NFR-12: one developer plus agents.
- The collector (`fdc`) is CPU-, IO- and disk-heavy: a 12-year backfill, a store of tens to hundreds of GB (M0 measures it), Arelle. The web app is light (long-term investing, low request rate, A-21).
- Hetzner raised cloud prices in June 2026. On 2026-11-06 its cloud product pages showed CX/CAX/CPX/CCX as "currently unavailable" ([Hetzner cost-optimized](https://www.hetzner.com/cloud/cost-optimized/), [regular performance](https://www.hetzner.com/cloud/regular-performance/), [general purpose](https://www.hetzner.com/cloud/general-purpose/); [wz-it.com, 2026-10-01](https://wz-it.com/en/blog/hetzner-price-increase-june-2026-cpx-ccx-alternatives/)). Dedicated AX servers were listed with prices ([matrix-ax](https://www.hetzner.com/dedicated-rootserver/matrix-ax/)).

## Options (monthly, EUR excl. VAT; v0.2 totals include marketstack Professional ≈ €43 — with the v0.3 Yahoo choice (ADR-012) subtract €43; detail in 03 §10)

| | A. Managed serving plane + Hetzner batch plane (**accepted**) | B. Lean: everything on one Hetzner VM (v0.1) | C. Scaleway VM + managed Postgres + Hetzner batch | D. All Scaleway (batch on a Scaleway VM) | E. AWS eu-central-1 (same shape) |
|---|---|---|---|---|---|
| Web / worker | Scaleway Serverless Containers (min 1 instance each) | systemd on CPX32 | systemd on Scaleway DEV1-L | Serverless Containers | ECS Fargate (min 1 each) behind an ALB |
| Tool runner | private Serverless Container, scale to zero, no credentials | systemd sandbox | systemd sandbox | private container | Fargate task / Lambda |
| App + market DB | Scaleway Managed PostgreSQL | SQLite + Litestream | Scaleway Managed PostgreSQL | Managed PostgreSQL | RDS PostgreSQL (db.t4g.micro/small) |
| Collector, reconciliation | Hetzner batch box (systemd) | same VM | Hetzner batch box | Scaleway DEV1-L + block storage | EC2 + EBS, or keep Hetzner |
| Email / observability | Scaleway TEM / Cockpit | EU provider / self | TEM / Cockpit | TEM / Cockpit | SES / CloudWatch Logs |
| **Total incl. price vendor** | **≈ €121–164** (plan €141) | ≈ €88–111 | ≈ €114–160 | ≈ €145–165 | **≈ €180–200 (estimate)** |
| Managed share | high (DB, compute, storage, email, logs) | none | DB, storage, email | highest | highest |
| Fit with owner preference | yes, the stated mix | no | partial | yes, but costs more than A for batch | yes, but ≈ €40–60/month more than A |

Option E (added v0.3 at the owner's question "is AWS cheaper?"): **coordinator estimate, not priced line by line**, ≈ €40–60/month more than A for the same set-up. The extra cost comes mainly from fixed charges that A does not have. An ALB costs $0.027/ALB-hour in Frankfurt ≈ $19.7/month before LCUs ([cloudburn.io, 2026-09](https://cloudburn.io/blog/aws-elastic-load-balancing-pricing)). Public IPv4 is $0.005/address-hour ≈ $3.65/month each, and Fargate tasks or NAT need several (same source). CloudWatch Logs ingestion is charged per GB. RDS and Fargate always-on minimums are also higher than Scaleway's DB-DEV-S and container free tier. No EU-sovereignty or cost benefit for this workload.

## Decision (accepted)
**Option A** (owner, 2026-11-06). Two planes:
1. **Serving plane: Scaleway, region fr-par.** nl-ams is acceptable if Scaleway's product-availability guide confirms Serverless Containers, Managed PostgreSQL and TEM there. Not verified, so Paris is the default. Components: `iip-web` and `iip-worker` as Serverless Containers; `iip-tool-runner` as a private container; Managed PostgreSQL (ADR-007); Object Storage for the published store snapshot (ADR-019); Transactional Email (ADR-017); Cockpit for logs and metrics; Container Registry; Secret Manager. Cloudflare stays in front (DNS, WAF, Turnstile; free plan).
2. **Batch plane: Hetzner, DE/FI.** One VM runs `fdc` (backfill, feed, watchers, FFIEC), `iip reconcile` and `fdc publish` under systemd, the v0.1 operating model. It holds the as-reported store of record on local NVMe or a volume, and Hetzner Object Storage holds the raw zips, export bundles and off-provider DB dumps (ADR-018). For the initial backfill, rescale the VM up for the backfill days (hourly billing, monthly cap), then back down.
3. **Fallback:** if spike S-1 fails, use **C**. The DB, storage, email and batch plane are unchanged; only web, worker and runner move to a Scaleway instance with systemd. S-1 fails if an always-on worker is throttled, cold starts break NFR-03, the private runner cannot be isolated from the database, or the measured cost exceeds the estimate by more than 50 %.

Batch box sizing: Hetzner **CX43** (8 vCPU/16 GB/160 GB, €16.49) when orderable, else **CPX32** (4/8/160, €35.99), else dedicated **AX41** (6-core/64 GB/2×512 GB NVMe, €59, no setup fee). The final size comes from M0.

## Consequences
+ Managed DB with provider backups, managed compute that restarts and scales itself, managed email and logs. Fewer things for one developer to patch (NFR-12).
+ A batch outage on Hetzner does not take the site down: the last published snapshot keeps serving, and only new filings are delayed (NFR-04 allows 24 h).
+ Fits €200 with headroom (≈ €40–80) for HA, a bigger DB node or a higher price-vendor tier.
− Two providers: cross-provider traffic (batch → Scaleway DB and Object Storage over TLS with an IP allow-list) and two consoles and bills.
− The systemd sandbox of v0.1 is replaced by container isolation plus "no credentials, read-only snapshot" for the runner (06 §3.3). Network egress from the runner container cannot be proven blocked (not verified). Accepted because the DSL performs no I/O (G-16).
− Some line items are unverified: container registry and Secret Manager prices, Hetzner volume price, Scaleway IPv4 price, whether Serverless Containers can attach to a Private Network, and Managed DB IP allow-lists and PITR. S-1 confirms them before provisioning.
