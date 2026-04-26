# methodology.json — manifest schema

The file `companies/{id}/methodology.json` is the install-state baseline for
[methodology-bootstrap](../SKILL.md). It is written by `install` (and by the
future `upgrade` slice landing in [COPAAA-69](/COPAAA/issues/COPAAA-69)) and
read read-only by the future `drift-check` slice landing in
[COPAAA-79](/COPAAA/issues/COPAAA-79). The CEO authorized writes to this
specific path outside `agents/` per Q3 of the COPAAA-58 plan-gate ratification.

The schema is reproduced verbatim from
[COPAAA-58 plan §0.2](/COPAAA/issues/COPAAA-58#document-plan) (rev 2) so a
maintainer reading this reference file alone has the full manifest contract.

## Schema (closed field set)

```json
{
  "installedVersion": "v1.0.0",
  "installedAt": "2026-04-26T08:00:00Z",
  "repoUrl": "https://github.com/jacobfogg/CopperForge-Method",
  "sourceCommitSha": "<40-char commit SHA the tag resolved to>",
  "files": [
    { "path": "<destination-relative-to-companies/{id}/>", "sha256": "<hex>" }
  ]
}
```

| Field | Type | Source | Notes |
|---|---|---|---|
| `installedVersion` | semver-style tag string (e.g. `v1.0.0`) | the `version` arg passed to install | Literal value the caller passed; never resolved or rewritten. |
| `installedAt` | ISO-8601 UTC timestamp | wall-clock at install time | Records when the manifest was written, not when the tag was cut. |
| `repoUrl` | string | constant `https://github.com/jacobfogg/CopperForge-Method` | Drift-check uses this to resolve the source on re-fetch; switching the canonical repo would be a coordinated change here. |
| `sourceCommitSha` | 40-char hex string | the SHA the version tag resolved to (workflow step 3) | Idempotency comparison key; force-tag re-pointing is detected via this field, not via `installedVersion`. |
| `files` | array of `{path, sha256}` | every destination written in workflow step 7 | `path` is **relative to `companies/{id}/`** (e.g., `COMPANY.md`, `agents/{agentId}/instructions/AGENTS.md`). `sha256` is the hex digest of the response body fetched from `raw.githubusercontent.com`. |

## Validation rules

- The five top-level fields above are the **closed** field set. Extra
  top-level fields are gate-failing under T6 of the install Test Gate.
- Every entry in `files[]` carries exactly the two keys `path` and `sha256`;
  extra per-entry keys are gate-failing.
- The `sourceCommitSha` field is exactly 40 hex chars (lowercase). The
  GitHub API returns lowercase commit SHAs; install does not transform.
- `files[].path` strings do NOT carry a leading `/` and do NOT carry the
  literal `companies/{id}/` prefix — they are pre-stripped to the relative
  form so drift-check can join them against the on-disk root without any
  transformation.

## Why the manifest is written last

Workflow step 9 writes the manifest **after** every per-file write succeeds.
The justification is partial-install recovery: if any per-file fetch or
write fails after step 7 begins, the manifest is NOT updated, and drift-check
continues to read the prior install's manifest. The next install attempt
still has the prior baseline as truth and surfaces the partial state as
drift on its first `drift-check` run.

## Idempotency comparison key

Workflow step 8 compares `sourceCommitSha` (not `installedVersion`) to decide
whether to short-circuit. The reason: a force-tagged release (operator
re-points `v1.0.0` at a different commit) needs to re-write every file, and
`installedVersion` would be unchanged. `sourceCommitSha` is the actual
content fingerprint and the correct comparison key.
