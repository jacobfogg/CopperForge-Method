# Drift-check output — comment + follow-up-issue templates

This reference reproduces the closed output contract that
[methodology-bootstrap](../SKILL.md) `drift-check` produces. It is the
single point of update when the drift-report shape changes; the SKILL.md
drift-check workflow steps reference this file rather than inline-duplicating
the comment / issue templates.

The contract is verified against the BA research summary at
[COPAAA-59 §2](/COPAAA/issues/COPAAA-59#document-research-summary)
(rev 1, 2026-04-26) and the COPAAA-58 plan §3 B3 + §0 R8 (rev 2,
CEO-ratified 2026-04-26).

## What drift-check writes (closed list)

A drift-check run writes **exactly** the artifacts in the table below — no
others. Writing under `companies/{id}/` (including any rewrite of
`methodology.json`) is gate-failing on T1; sending an approval / PATCHing
another issue / creating a routine on a drift-check run is gate-failing on
T4 and T8. The reporter integration in [COPAAA-61](/COPAAA/issues/COPAAA-61)
is a future hook; v1 drift-check posts to the originating issue thread only
per [COPAAA-58 plan §0](/COPAAA/issues/COPAAA-58#document-plan) R8.

| Branch | Comment on originating issue | Follow-up issue | Files written | Approvals posted |
|---|---|---|---|---|
| No drift (T4) | One quiet comment (`no drift detected` template below) | none | none | none |
| Drift detected (T5/T6/T7) | One drift-report comment (template below) | One follow-up issue (template below) | none | none |

## Drift-entry shape (used by both branches)

The drift accumulator built in SKILL.md drift-check workflow step 3 + step 4
holds a list of entries with the closed shape:

```
{
  change: "modified" | "deleted" | "unexpected",
  path: "<destination-relative-to-companies/{id}/>",
  manifestSha256: "<40-char hex>" | null,
  onDiskSha256: "<40-char hex>" | null
}
```

| `change` | When | `manifestSha256` | `onDiskSha256` |
|---|---|---|---|
| `modified` | manifest entry exists, file exists, sha256 differs | manifest's `entry.sha256` | hash of on-disk bytes |
| `deleted` | manifest entry exists, file is missing on disk | manifest's `entry.sha256` | `null` |
| `unexpected` | file exists in an instrumented dir, no manifest entry matches | `null` | hash of on-disk bytes |

Drift entries MUST be sorted in this order before rendering: by `change`
in the order `modified, deleted, unexpected`, then by `path` lexicographically.
Stable ordering keeps the audit trail diffable across runs.

## No-drift comment (T4)

Posted on the originating issue. Exactly one comment, no follow-up issue,
no other side effects. The literal phrase `methodology-bootstrap drift-check: no drift detected` MUST appear so QA2's T4 can pattern-match it.

```markdown
## methodology-bootstrap drift-check: no drift detected

**Manifest baseline:** `{installedVersion}` / sha `{sourceCommitSha}`
**Manifest installed at:** `{installedAt}`
**Files compared:** `{files.length}` manifest entries
**Result:** clean — every manifest entry's on-disk sha256 matches the recorded baseline; zero unexpected files in instrumented directories.
```

The four facts (`installedVersion`, `sourceCommitSha`, `installedAt`,
`files.length`) come straight from the manifest read in SKILL.md
drift-check workflow step 2. Free-text padding is allowed; omission of any
of the four facts is gate-failing.

## Drift-report comment (T5/T6/T7)

Posted on the originating issue when the drift accumulator is non-empty.
The literal phrase `methodology-bootstrap drift-check: drift detected` MUST
appear so QA2 can pattern-match. The comment includes the drift table and
the link out to the follow-up issue (whose identifier is captured before
the comment is posted; see SKILL.md drift-check workflow step 6.3).

```markdown
## methodology-bootstrap drift-check: drift detected

**Manifest baseline:** `{installedVersion}` / sha `{sourceCommitSha}`
**Manifest installed at:** `{installedAt}`
**Drift summary:** `{modifiedCount}` modified, `{deletedCount}` deleted, `{unexpectedCount}` unexpected (total `{totalDriftEntries}`)
**Follow-up issue:** [`{followUpIssueIdentifier}`](/{prefix}/issues/{followUpIssueIdentifier})

### Drift entries

| change | path | manifest sha256 | on-disk sha256 |
|---|---|---|---|
| `{change}` | `{path}` | `{manifestSha256 or "(not tracked)"}` | `{onDiskSha256 or "(missing)"}` |
| ... | ... | ... | ... |
```

Rendering rules:

- One row per drift accumulator entry, sorted per the entry-shape rule above.
- `manifestSha256: null` renders as the literal string `(not tracked)`.
- `onDiskSha256: null` renders as the literal string `(missing)`.
- The full sha256 (40 hex chars, lowercase) is rendered verbatim — do **not**
  truncate. Auditor challenges depend on byte-exact comparability across
  drift-check runs.
- The `Drift summary` line uses the literal counts; do not pluralize "entries"
  if the count is 1, except in the follow-up issue title (see below).

## Follow-up issue (T5/T6/T7)

Created via `POST /api/companies/{companyId}/issues` when the drift
accumulator is non-empty. Assignee is the resolved CEO (the same agent
that passed the caller-identity check). Priority is `medium`. The phrase
`methodology-bootstrap drift detected` MUST appear in both the title and
the description.

### Title template

```
methodology-bootstrap drift detected on {companyId} ({totalDriftEntries} entr{y/ies})
```

The trailing `entr{y/ies}` is `entry` when `totalDriftEntries == 1` and
`entries` otherwise. The `{companyId}` is the target company UUID
(matches `payload.companyId` for symmetry with the upgrade approval payload
in [`approval-flow.md`](approval-flow.md)).

### Description template

```markdown
## methodology-bootstrap drift detected

A `drift-check` run on company `{companyId}` against manifest baseline `{installedVersion}` / sha `{sourceCommitSha}` (installed `{installedAt}`) found `{totalDriftEntries}` drift entr{y/ies}: `{modifiedCount}` modified, `{deletedCount}` deleted, `{unexpectedCount}` unexpected.

**Originating issue:** [{originatingIssueIdentifier}](/{prefix}/issues/{originatingIssueIdentifier})

### Drift entries

| change | path | manifest sha256 | on-disk sha256 |
|---|---|---|---|
| `{change}` | `{path}` | `{manifestSha256 or "(not tracked)"}` | `{onDiskSha256 or "(missing)"}` |
| ... | ... | ... | ... |

### Recommended responses (choose one)

1. **Revert to canonical.** Re-install the methodology at `{installedVersion}` to restore tracked files to the baseline recorded in `companies/{companyId}/methodology.json`. This is `op=install version={installedVersion}` against the same manifest's `repoUrl`. Discards any in-place edits captured under `modified`; deletes any file captured under `unexpected` is **not** automatic — the operator must remove unexpected files separately.

2. **Formalize as a methodology-repo PR.** Capture the `modified` and `unexpected` content as a PR against `{repoUrl}` (the canonical methodology repo) so that the next tagged release includes them. After the PR merges and a new tag is cut, run `op=upgrade target={newTag}` on this company to align on-disk content with the new manifest.

These are the **two** recommended responses. Drift-check intentionally does NOT recommend a third option such as "upgrade to a newer existing version" — that is the `upgrade` operation's concern (see [COPAAA-69](/COPAAA/issues/COPAAA-69)), and conflating drift with version-availability encourages operators to re-install on top of in-place edits without first deciding whether those edits should be discarded or formalized.
```

The `{prefix}` value is derived from the originating issue's identifier
(e.g., `COPAAA-NN` → prefix `COPAAA`) and reused for both the originating-issue
back-link and any future cross-references the operator adds.

## Why a separate file

The drift-check SKILL.md workflow reduces to seven numbered steps once the
read-only invariant, the drift-entry shape, and the two-branch output
contract are reproduced; inlining the comment / issue templates would push
the file past the [research-summary](/COPAAA/issues/COPAAA-15#document-research-summary)
quality bar for SKILL.md length and operability. The pattern matches the
[gate-review/references/{code-gate.md, disposition-template.md}](/COPAAA/issues/COPAAA-49)
split: material referenced by but not part of the operational workflow goes
to a sibling reference file, and the SKILL.md cites the reference at the
exact step that consumes it.
