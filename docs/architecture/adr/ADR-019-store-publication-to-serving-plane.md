# ADR-019: Publishing the as-reported store from the batch plane to the serving plane

- Status: Accepted (new in v0.2, follows ADR-015); details validated by spike S-1
- Date: 2026-11-06
- Deciders: architect (design mode)

## Context
In v0.2 the collector and the as-reported store of record (catalog + per-company SQLite, ADR-003) run on the Hetzner batch box. Web, worker and tool-runner run as managed containers on Scaleway with no shared disk (ADR-015). Company pages need statements as presented, notes and revisions (FR-08, UC-01) at p95 < 2 s (NFR-03). Built-in and user tools need standardized financials and prices for the whole universe (FR-10, FR-11). The serving plane must keep working when the batch box is down (NFR-06). The store is public data (SEC/FFIEC); no personal data is involved.

## Options
1. Expose `store_client` as an HTTPS API on the batch box. Simple, but a batch outage takes company pages down, and the batch box becomes part of the serving path.
2. Load the whole as-reported store into managed Postgres. Large (FSN notes), costly DB storage, and it duplicates the store's role.
3. **Immutable published snapshots on object storage**, plus small indexes in Postgres:
   - after each load, `fdc publish` uploads changed per-company files (`companies/<id>/<content-hash>.sqlite`) and a new `catalog-<seq>.sqlite` to Scaleway Object Storage (one-zone, bucket `iip-store`), then appends `market.filing_event` rows and updates `market.company_store_version(company_id, object_key, seq)` in Postgres;
   - `iip reconcile` (batch box) writes `market.std_financials`/`std_lineage` directly to Postgres (role `iip_batch`);
   - the web and worker containers read per-company files through a `FilingStore` adapter that downloads by object key into a bounded on-disk LRU cache (immutable keys, so no invalidation is needed) and opens them read-only;
   - the worker builds an **analysis snapshot** (`analysis-<seq>.sqlite`: std_financials + latest prices + company index) after every reconcile or price refresh; the tool runner receives a short-lived pre-signed GET URL for it in each run request and caches it by `seq`.

## Decision
Option 3. Published keys are content-addressed and never overwritten. Old versions are kept 30 days (lifecycle rule), so a page or run can always finish on the version it started with. `store_schema_version` stays the published-language contract (contract tests, 04 §5).

## Consequences
+ The serving plane does not depend on the batch box being up: the last published state keeps serving, and NFR-04 tolerates a delay of up to 24 h.
+ The per-company SQLite format is unchanged, so FR-05 export bundles and dev seeds work as in v0.1.
+ Object storage cost is small (one-zone €0.00752/GB-month; 100 GB ≈ €0.75; [Scaleway](https://www.scaleway.com/en/pricing/storage/)).
− The first view of a cold company pays one object download (est. 50–300 ms for a 1–10 MB file in-region, **not measured**). Mitigation: warm the cache for watchlist and portfolio companies at container start. Measured in S-1 against NFR-03.
− Container ephemeral disk size is **not verified**. If it is too small, keep only hot companies cached and fall back to HTTP range reads.
− Cross-provider egress: Hetzner → Scaleway ingress is free on Scaleway's side, and Hetzner egress is included in its 20 TB/VM allowance.
