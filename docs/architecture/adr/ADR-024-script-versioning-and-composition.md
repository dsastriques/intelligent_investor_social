# ADR-024: Script versioning and composition: immutable content-hashed versions, exact pins, vendored closures

- Status: Proposed (architect, design v0.7; pin-permission rule is owner question Q-19)
- Date: 2026-11-07
- Deciders: architect (design mode); owner for Q-19

## Context
BR-22: scripts are versioned and may call other scripts; a call pins the called version, and a new version never changes the results of existing dependents. BR-06/BR-07: private by default, shared per item and audience, and recipients may copy (ASSUMPTION A-R9). FR-19: sharing can be revoked. FR-27/BR-19: deleting an account deletes its private items. Built-ins are code in the repo (FR-32).

## Options
1. Pins by reference only (dependent stores `script_id@version`). Simple, but when the source is unshared or deleted, every dependent breaks, which in effect changes its results.
2. **Pins by reference plus a vendored closure:** on publish, the dependent stores the full resolved closure (spec JSON and content hashes of every transitive dependency).
3. Inline expansion: copy the dependency's formula into the dependent's formula. Loses structure and trace readability.

## Decision
Option 2 (08 §5):
- `ScriptVersion` is immutable: no update path, DB trigger on `tl_script_version`. `content_hash` = SHA-256 of the canonical spec, the closure hashes and, for built-in Python, entrypoint + `code_version` + screener package version.
- `uses: [{as, script, version}]` is an exact pin, never "latest". Only published versions can be pinned, so the graph is a DAG by construction. DFS cycle and limit checks (depth ≤ 8, size ≤ 32) run anyway.
- Effective inputs are unified across the closure by `(concept, basis, series)`. Conflicts are save-time errors.
- **Vendored closure** stored with each dependent version, so revoking a share or deleting the author's account does not change or break dependents. This matches the copy semantics of BR-07.
- Pinning another user's shared script requires the `copy` permission, because it vendors the formula (Q-19). Built-ins are pinnable by all.
- **Built-in versions** are loaded idempotently at deploy by content hash. A changed spec or implementation under an existing version number fails CI and the deploy. Golden fixtures per built-in version (G-29).
- Sharing, revoking and copying use the existing `Share` model, with item type `script` added to the tool rows of the policy (06 §2.2).

## Consequences
+ BR-22 holds by construction and is testable: re-running any stored ScriptRun's closure with its trace inputs gives byte-identical outputs.
+ The GDPR deletion of a source does not break other users' work, the same way copied tools stay theirs (06 §4.2).
− A vendored closure keeps a copy of a deleted user's formula inside other users' scripts. This is the same as an accepted copy (BR-07). Stated in the privacy notice next to tool copies (R-22).
− No automatic propagation of dependency fixes: users upgrade dependents explicitly. The UI shows "newer version available".
