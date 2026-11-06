# ADR-018: Raw data retention in object storage, not GitHub

- Status: Accepted; provider per Q-11
- Date: 2026-11-06
- Deciders: architect (design mode); owner for items marked Proposed

## Context
Research: raw SEC zips too big for GitHub; the store must be rebuildable (NFR-06) and seedable (FR-05).

## Options
1. Keep zips only on server disk. 2. Server cache + EU object storage as the durable copy. 3. Git LFS.

## Decision
Option 2: `RawArchive` port with local cache + S3-compatible bucket (EU); checksums in `collected_item`; `.gitignore` excludes data dirs; weekly `fdc export` bundles stored alongside.

## Consequences
+ store can be rebuilt without re-downloading from SEC; cheap storage.
− object storage cost (≈ €2–10/month) and credentials to manage.
