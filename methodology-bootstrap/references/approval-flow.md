# Approval flow — `upgrade` operation

This reference reproduces the full board-approval contract that
[methodology-bootstrap](../SKILL.md) `upgrade` drives. It is the single
point of update when the Paperclip approval API changes shape; the SKILL.md
upgrade workflow steps reference this file rather than inline-duplicating
the payload schema.

The flow is verified against the BA research summary at
[COPAAA-59 §2C and §3.3](/COPAAA/issues/COPAAA-59#document-research-summary)
(rev 1, 2026-04-26) and the COPAAA-58 plan §0/§3 (rev 2, CEO-ratified
2026-04-26).

## Lifecycle (two heartbeats)

`upgrade` is split across **two** CEO heartbeats. The skill must not poll
the approvals API; Paperclip's `heartbeat.wakeup()` is fire-and-forget into
a fresh heartbeat once the board acts.

```
heartbeat 1 (request mode)
  ├── caller-identity check (CEO-only)
  ├── target-arg check (no "latest")
  ├── prior-install check (methodology.json must exist)
  ├── resolve target tag → toSha (anonymous GitHub)
  ├── fetch both methodology trees, compute diffSummary
  ├── upload full unified diff as issue-document
  ├── POST /api/companies/{id}/approvals (kind=methodology_upgrade)
  ├── comment "approval requested, awaiting board"
  └── EXIT (do NOT poll)
                  ⋮ board reviews ⋮
                  ⋮ board approves or rejects ⋮
                  ⋮ Paperclip wakes the requesting CEO ⋮
heartbeat 2 (apply mode, fired by approval-resolution wake)
  ├── apply-mode env check (PAPERCLIP_APPROVAL_ID/STATUS/LINKED_ISSUE_IDS)
  ├── status branch (approved → apply; rejected → rejection path)
  ├── caller-identity re-check (defense-in-depth)
  ├── read approval (verify kind == methodology_upgrade)
  ├── idempotency check (manifest.sourceCommitSha == payload.toSha → no-op)
  ├── stage-then-atomic-rename file write
  ├── manifest write (last)
  └── audit-trail comment
```

## Environment-variable contract (apply mode)

Apply mode is reachable **only** when all three of the following heartbeat
env vars are set. Any missing or malformed value MUST cause the skill to
reject with the literal phrase `apply mode requires approval wake env`
(SKILL.md upgrade apply step 1).

| Env var | Value | Source |
|---|---|---|
| `PAPERCLIP_APPROVAL_ID` | The UUID of the resolved approval | `heartbeat.wakeup()` payload `approvalId` (server `routes/approvals.js:127–148`) |
| `PAPERCLIP_APPROVAL_STATUS` | Literal `approved` or `rejected` | `heartbeat.wakeup()` payload `approvalStatus` |
| `PAPERCLIP_LINKED_ISSUE_IDS` | CSV of issue UUIDs that include the originating issue id from the approval's `issueIds[]` | `heartbeat.wakeup()` payload `issueIds` |

The skill does **not** rely on a same-heartbeat return value from `POST /api/companies/{id}/approvals`. The approval id is kept ambient
(captured in the request-mode comment) so a forensic re-fire of apply mode
remains tractable without mutable session state.

## Approval payload — closed shape

POST body for `POST /api/companies/{companyId}/approvals` in request mode:

```json
{
  "type": "request_board_approval",
  "requestedByAgentId": "<CEO agent UUID>",
  "issueIds": ["<originating issue UUID>"],
  "payload": {
    "kind": "methodology_upgrade",
    "companyId": "<target company UUID>",
    "fromVersion": "v1.0.0",
    "toVersion": "v1.1.0",
    "fromSha": "<40-char from manifest.sourceCommitSha>",
    "toSha": "<40-char from GitHub commits/{target}>",
    "summary": "<one-paragraph board-readable summary of what changes>",
    "diffSummary": [
      {
        "file": "methodology/CODING_STANDARDS.md",
        "change": "modified",
        "linesAdded": 12,
        "linesRemoved": 3
      }
    ],
    "fullDiffArtifactUrl": "<issue-document URL with the unified diff>",
    "risks": [
      "<auditor-style risk note 1>"
    ],
    "rollbackVersion": "v1.0.0"
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `type` | string | MUST be the literal `request_board_approval` (closed enum, `shared/dist/constants.js:157–162`). |
| `requestedByAgentId` | UUID | The caller's CEO agent id; must match the agent the approval-resolution wake is delivered to. |
| `issueIds` | UUID[] | Includes the originating issue id so apply-mode wake carries it through `PAPERCLIP_LINKED_ISSUE_IDS`. |
| `payload.kind` | string | MUST be the literal `methodology_upgrade`; apply mode rejects approvals with any other `kind`. |
| `payload.companyId` | UUID | The target company. Carried forward into apply mode without re-derivation. |
| `payload.fromVersion` / `payload.toVersion` | tag string | Both literal tags. Rollback is `toVersion < fromVersion` per semver, not a separate kind. |
| `payload.fromSha` / `payload.toSha` | 40-char hex | `fromSha` is `manifest.sourceCommitSha` at request time; `toSha` is the resolved target commit. |
| `payload.summary` | string | Board-readable. Should name the operation, the version pair, and the headline file changes. |
| `payload.diffSummary[]` | object[] | One entry per changed file. `change ∈ {added, modified, removed}`. `linesAdded`/`linesRemoved` are integers; missing one is gate-failing on T9. |
| `payload.fullDiffArtifactUrl` | URL | Required. The issue-document URL of the uploaded unified diff. **Never** inline the full diff in `payload`. |
| `payload.risks[]` | string[] | Auditor-style risk list. May be empty for low-risk patch upgrades. |
| `payload.rollbackVersion` | tag string | MUST equal `manifest.installedVersion` at the time of request (i.e., the version the operator can fall back to without further action). |

## Payload size

The full unified diff lives behind `fullDiffArtifactUrl`, **not** inline.
Justification: methodology releases can include large CODING_STANDARDS or
PLATFORM diffs; embedding them in the approval payload risks exceeding
practical board-review surface and (per BA R10) the approvals API isn't a
diff store. The artifact is created via
`POST /api/issues/{originating-issue-id}/documents` with key
`methodology-upgrade-diff`. Subsequent re-requests for the same issue MUST
post a new revision of the same document key rather than creating a fresh
document, so the artifact URL remains stable per originating issue.

## Atomic apply (R2)

Apply mode follows a stage-then-rename pattern to make partial-failure
recovery deterministic:

1. **Stage.** Create `companies/{id}/.methodology-upgrade-{toSha}/`. Re-fetch every target-side raw blob (15 reads) and write under the staging dir mirroring the destination map produced by the role-fanout (3 company-root paths + N×per-role paths).
2. **Rename.** For each destination path, atomic-rename from staging onto the live tree. Per-agent `instructions/` is renamed leaf-by-leaf (or as a directory rename, whichever the runtime supports atomically); company-root files are renamed onto `companies/{id}/`.
3. **Manifest.** Only after every rename succeeds is `companies/{id}/methodology.json` rewritten with the new `sourceCommitSha`, `installedVersion`, `installedAt`, and a recomputed `files[]` array of `{path, sha256}`.
4. **Cleanup.** Remove the staging dir on full success. On any rename failure, leave the staging dir in place — the next `drift-check` invocation surfaces the partial state and the operator can either re-run apply (which is idempotent) or roll back via a new approval.

The invariant is: `methodology.json` always reflects a fully-applied SHA;
a state where the manifest claims `toSha` but a file under `companies/{id}/`
still has the `fromSha` content is structurally impossible without an
out-of-band edit (which is exactly what `drift-check` is meant to detect).

## Idempotency (R5)

Apply mode is re-entrant on the same approval. If `heartbeat.wakeup()` is
delivered twice (e.g., a runtime retry after a partial heartbeat failure),
the second wake's idempotency short-circuit triggers because
`manifest.sourceCommitSha == payload.toSha` after the first wake's manifest
write. The audit-trail comment in the second wake uses the literal phrase
`already at sourceCommitSha {sha}; no files written` so the audit trail
distinguishes a no-op replay from a fresh apply.

The comparison key is `sourceCommitSha`, not `installedVersion`, for the
same reason as the install workflow: a force-tagged release re-points a
tag at a different commit and `installedVersion` would not change.

## Rejection path

When `PAPERCLIP_APPROVAL_STATUS == "rejected"`, apply mode reads the
approval's rejection reason (`approval.decisionReason` or whichever field
the approvals API exposes — the skill is field-tolerant on read) and posts
a single comment on the originating issue containing the literal phrase
`methodology-bootstrap upgrade rejected` plus the rejection reason and
the originally-requested `fromVersion → toVersion`. **No** files under
`companies/{id}/` are touched, including no manifest update. The audit
trail (request-mode comment + rejection comment + approval record) is the
sole evidence that an upgrade was attempted and refused; T7 of the upgrade
Test Gate verifies zero file changes on this path.

## Why a separate file

The upgrade SKILL.md workflow reduces to ~20 numbered steps once both
modes plus the rejection path are reproduced; inlining the payload shape
and the env-var contract would push the file past the
[research-summary](/COPAAA/issues/COPAAA-15#document-research-summary)
quality bar for SKILL.md length and operability. The pattern matches the
[gate-review/references/{code-gate.md, disposition-template.md}](/COPAAA/issues/COPAAA-49)
split: material referenced by but not part of the operational workflow
goes to a sibling reference file.
