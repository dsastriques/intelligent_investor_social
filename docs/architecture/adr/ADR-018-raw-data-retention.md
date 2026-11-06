# ADR-018: Raw data retention in object storage, not GitHub

- Status: Accepted; provider **Hetzner Object Storage** (follows ADR-015 option A, accepted by the owner 2026-11-06; Q-11 formally still open)
- Date: 2026-11-06 (v0.2)
- Deciders: architect (design mode); owner for the provider

## Context
Research: raw SEC zips are too big for GitHub. The store must be rebuildable (NFR-06) and seedable (FR-05). In v0.2 the batch plane runs on Hetzner and the serving plane on Scaleway (ADR-015).

## Options
1. Keep zips only on the server disk. 2. Server cache plus EU object storage as the durable copy. 3. Git LFS.
Provider for 2: (a) Hetzner Object Storage, €6.49/month including 1 TB storage and 1 TB egress, ingress free; FSN1/NBG1/HEL1 ([Hetzner](https://hetzner.com/storage/object-storage)). (b) Scaleway Object Storage, €0.0146/GB-month multi-AZ, €0.00752 one-zone, 75 GB egress free ([Scaleway](https://www.scaleway.com/en/pricing/storage/)). (c) Cloudflare R2 with EU jurisdiction (price not re-verified in v0.2).

## Decision
Option 2 with **(a) Hetzner Object Storage** for the batch plane. The bucket `fdc-raw` holds raw zips (kept), `fdc-bundles` the weekly `fdc export` bundles (8 weeks), and `iip-dumps` the nightly encrypted Postgres dumps (35 days, ADR-007). Behind it sits the `RawArchive` port: a local cache plus the S3-compatible bucket. Checksums go in `collected_item`, and `.gitignore` excludes data directories.
The **serving** copy of the store lives on Scaleway Object Storage one-zone (ADR-019). It is rebuildable, so one-zone is enough.

## Consequences
+ Same provider and network as the batch box: free internal traffic, a flat €6.49 that covers the expected ≤ 1 TB.
+ Backups of personal data (dumps) sit with a different provider from the primary DB, and both are in the EU.
− A second set of credentials, scoped per bucket. Dumps are encrypted client-side (`age`), so the storage provider sees no plaintext personal data.
