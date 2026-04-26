---
name: methodology-bootstrap
description: >
  Install or upgrade a pinned CopperForge methodology release (v1.0.0+ tag of
  jacobfogg/CopperForge-Method) on a target Paperclip company. The `install`
  operation fetches the methodology/ tree at the resolved 40-char commit SHA,
  fans out per-role files to every matching agent's instructions/ directory,
  writes company-root files once at companies/{id}/, and records install
  state in companies/{id}/methodology.json. The `upgrade` operation adds a
  mandatory board-approval gate before any file change: request mode posts
  a `request_board_approval` with a diff summary (full unified diff linked
  out as an issue-document artifact) and exits the heartbeat; apply mode
  resumes in a fresh heartbeat woken by the approval-resolution env vars
  (`PAPERCLIP_APPROVAL_ID`, `PAPERCLIP_APPROVAL_STATUS`,
  `PAPERCLIP_LINKED_ISSUE_IDS`) and writes files atomically (staging rename,
  manifest written last). Rollback is the same code path with a lower target
  version. Use this skill when the CEO of a target Paperclip company needs
  to install a tagged methodology release for the first time or upgrade
  (or roll back) between two pinned tag versions. v1 ships `install` and
  `upgrade`; the read-only `drift-check` operation lands in COPAAA-79.
---

# methodology-bootstrap (install, upgrade)

Use this skill when you are the **CEO of a target Paperclip company** and need to install or upgrade a pinned tagged release of the CopperForge methodology repo (`jacobfogg/CopperForge-Method`) on that company. Both operations share the same caller-identity check (`role=ceo` resolved against the target company's `/agents` response per [`references/role-normalization.md`](references/role-normalization.md)), the same anonymous GitHub fetcher pattern, the same role-fanout destination map, and the same `companies/{id}/methodology.json` schema (see [`references/manifest-schema.md`](references/manifest-schema.md)). They diverge in two places: `install` writes immediately on first invocation, while `upgrade` is split into a **request mode** (post a `request_board_approval`, exit the heartbeat) and an **apply mode** (resumed in a fresh heartbeat by the approval-resolution wake — see [`references/approval-flow.md`](references/approval-flow.md) — which writes files atomically and updates the manifest). Re-running either operation at the already-installed `sourceCommitSha` is a safe no-op (manifest comparison short-circuits writes). The read-only `drift-check` operation lands with [COPAAA-79](/COPAAA/issues/COPAAA-79).

## Preconditions

- The caller passes an `op` argument with the value `install` or `upgrade`. Any other value (including missing or `"drift-check"` until [COPAAA-79](/COPAAA/issues/COPAAA-79) lands) MUST be rejected before any further work. The two operations share preconditions but have separate workflows below.
- The caller is the CEO of the target company. CEO identity is resolved by the role-enum normalization table in [`references/role-normalization.md`](references/role-normalization.md) (`role=ceo` rule) against the live `GET /api/companies/{id}/agents` response. Any other caller MUST be rejected before any disk read or network call. This is **non-negotiable #1 (CEO-only enforcement)** from the COPAAA-58 plan and is enforced at every operation, not just install. Apply mode of the upgrade operation re-runs this check inside the resumed heartbeat as a defense-in-depth against malformed wake env (T2 of the upgrade Test Gate).
- The pinning argument is present and is a literal tag identifier (e.g., `v1.0.0` or `v1.0.1`). For `op=install` the argument is named `version`; for `op=upgrade` (request mode) the argument is named `target`. The string `"latest"`, the empty string, and a missing argument MUST all be rejected before any network call. This is **non-negotiable #3 (version pinning)** from the COPAAA-58 plan; neither operation ever resolves "current" or "head" implicitly. Apply mode of the upgrade operation reads its target from the resolved approval payload (see [`references/approval-flow.md`](references/approval-flow.md)), not from a fresh argument, and is exempt from this precondition.
- The canonical methodology repo `https://github.com/jacobfogg/CopperForge-Method` is reachable anonymously over HTTPS. v1 issues no `Authorization` header on any GitHub call; the repo is public and the GitHub anonymous rate limit (60 req/h) covers the 17 reads a single install issues (1 commits, 1 tree, 15 raw blobs) and the ~19 reads a single upgrade request mode issues (2 commits, 2 trees, 15 raw blobs at the target SHA — current-SHA blobs are not refetched because the manifest carries the per-file `sha256` baseline).
- The target company id is known and is the company the skill writes into. Writes to disk happen under `companies/{id}/` only; the skill never touches agent records, company records, or any path outside the `companies/{id}/` subtree (with the single exception of `companies/{id}/methodology.json`, which the CEO has explicitly authorized as outside-`agents/` per Q3 of the Plan-Gate ratification).
- The role-enum normalization table in [`references/role-normalization.md`](references/role-normalization.md) is loaded. The two hard-fail rules in that table (`zero-match` and `multi-match`) MUST be enforced before any disk write by every operation that consumes the normalization table — both `install` and `upgrade` apply mode.
- For `op=upgrade` only: a prior install manifest at `companies/{id}/methodology.json` MUST already exist. Upgrade is not bootstrap; "install first" is a hard precondition (T3 of the upgrade Test Gate).

## Install workflow (op=install)

1. **Caller identity check (non-negotiable #1).** Read the caller's agent identifier from the invocation context (`PAPERCLIP_AGENT_ID` or equivalent). Resolve the target company's CEO by `GET /api/companies/{id}/agents` and applying the `role=ceo` rule from [`references/role-normalization.md`](references/role-normalization.md). If the caller's id does not match the resolved CEO's id, reject with the literal message `"methodology-bootstrap: CEO-only operation; caller {agent-id} is not the CEO of company {company-id}"` and exit before any further read or write. Free-text variants of this message ("only the CEO can run this", "not authorized", etc.) are malformed — the literal phrase `CEO-only` MUST appear so QA2's T2 can pattern-match it.

2. **Version-arg check (non-negotiable #3).** Read the `version` argument. If it is missing, the empty string, the literal string `"latest"`, or any value that does not parse as a tag identifier (a non-empty string with no whitespace), reject with the literal message `"methodology-bootstrap: version argument is required and must be a literal tag (e.g. v1.0.0); received {literal-value}"` and exit before any network call. The literal phrase `version argument is required` MUST appear so QA2's T1 can pattern-match it.

3. **Resolve tag → 40-char commit SHA.** `GET https://api.github.com/repos/jacobfogg/CopperForge-Method/commits/{version}` with `Accept: application/vnd.github+json` and no `Authorization` header. On HTTP 404 or any non-2xx response, reject with the literal message `"methodology-bootstrap: version tag {version} did not resolve on jacobfogg/CopperForge-Method; HTTP {status}"` and exit before any disk write. The resolved SHA is the commit object's `sha` field — exactly 40 hex chars; reject the response if the field is absent or the wrong length. The literal phrase `did not resolve` MUST appear so QA2's T3 can pattern-match it.

4. **Fetch the methodology tree at the resolved SHA.** `GET https://api.github.com/repos/jacobfogg/CopperForge-Method/git/trees/{sha}?recursive=1` (anonymous). Filter the returned `tree[]` array to entries whose `path` starts with `methodology/` and whose `type` is `blob`.

5. **Verify the 15 expected source files are present.** Match the filtered tree against the closed file list in [`references/source-files.md`](references/source-files.md) (3 company-root files + 12 per-role files = 15 total; `methodology/README.md` is explicitly optional per CTO §9.6 recommendation and is NOT required). If any of the 15 expected paths is missing, reject with the literal message `"methodology-bootstrap: methodology/ tree at {sha} is missing expected file(s): {comma-separated paths}"` and exit before any disk write. The literal phrase `missing expected file` MUST appear so QA2's T4 can pattern-match it.

6. **Build the destination map (template-fanout per Plan Q2/Q5).** Apply the role-enum normalization table in [`references/role-normalization.md`](references/role-normalization.md) against the live `GET /api/companies/{id}/agents` response to resolve every `methodology/roles/<Role>/` source directory to **all** matching agents. Then enforce the two hard-fail rules:
   - **Hard-fail rule 1 (zero-match).** Any methodology source dir resolving to zero agents → reject with the literal message `"methodology-bootstrap: methodology source dir(s) resolved to zero agents in company {company-id}: {comma-separated source dirs}"` and exit before any disk write. The literal phrase `resolved to zero agents` MUST appear so QA2's T9 can pattern-match it.
   - **Hard-fail rule 2 (multi-match).** Any single agent matching two or more methodology source dirs (i.e., the match rules overlap on that agent) → reject with the literal message `"methodology-bootstrap: agent {agent-id} matches multiple methodology source dirs: {comma-separated source dirs}"` and exit before any disk write. The literal phrase `matches multiple methodology source dirs` MUST appear so QA2's T10 can pattern-match it.

   Construct the destination map as a list of `{sourcePath, destinationPath}` pairs:
   - Each company-root file (`methodology/COMPANY.md`, `methodology/PLATFORM.md`, `methodology/CODING_STANDARDS.md`) → exactly one destination at `companies/{id}/{COMPANY,PLATFORM,CODING_STANDARDS}.md`. **NOT** replicated into every agent dir.
   - Each per-role file (`methodology/roles/<Role>/<file>.md`) → one destination per matching agent: `companies/{id}/agents/{agentId}/instructions/<file>.md`.

7. **Fetch + write each file.** For every destination map entry:
   1. `GET https://raw.githubusercontent.com/jacobfogg/CopperForge-Method/{sha}/{sourcePath}` (anonymous). On any non-2xx, reject with `"methodology-bootstrap: failed to fetch {sourcePath} at {sha}; HTTP {status}"` and exit (the install is atomic at the file-set level — partial writes are allowed only when the manifest is NOT written; see step 9).
   2. Compute `sha256` of the response body.
   3. Ensure the destination directory exists (`mkdir -p`).
   4. Write the response body to the destination path.
   5. Record `{path, sha256}` in the manifest's `files[]` accumulator, where `path` is **relative to `companies/{id}/`** (e.g., `COMPANY.md` or `agents/{agentId}/instructions/AGENTS.md`).

8. **Idempotency short-circuit (T8).** If a `companies/{id}/methodology.json` already exists AND its `sourceCommitSha` matches the SHA resolved in step 3, the install is a no-op: skip steps 7 and 9, post the audit-trail comment in step 10 with the literal phrase `"already at sourceCommitSha {sha}; no files written"`, and exit. The comparison MUST happen on `sourceCommitSha`, not on `installedVersion` (so re-installing `v1.0.0` after a force-tag points the tag at a different commit will still re-write).

9. **Write the manifest.** Write `companies/{id}/methodology.json` per the schema in [`references/manifest-schema.md`](references/manifest-schema.md):
   ```json
   {
     "installedVersion": "{version}",
     "installedAt": "{ISO-8601 UTC timestamp}",
     "repoUrl": "https://github.com/jacobfogg/CopperForge-Method",
     "sourceCommitSha": "{40-char SHA from step 3}",
     "files": [ { "path": "{relative}", "sha256": "{hex}" }, ... ]
   }
   ```
   The `files[]` array length equals `3 + sum_over_role_dirs(matching_agent_count_for_each_per_role_file)` per the Plan §0.1 normalization table — 3 company-root entries plus one entry per (agent × per-role-file) destination produced by step 6's fanout. The exact cardinality is roster-dependent; do NOT hard-code a length assertion. The manifest is the single source of truth for `drift-check`; write it last so a partial install (steps 1–7 succeed, step 9 fails) leaves no manifest and `drift-check` will continue to read the prior install's state.

10. **Post the audit-trail comment.** Post a single comment on the originating issue (the issue from which the install was invoked, identified by `PAPERCLIP_LINKED_ISSUE_IDS` or equivalent invocation context) containing exactly four facts: `installedVersion`, `installedAt`, file count (`files.length` from the manifest), and `sourceCommitSha`. The literal phrase `methodology-bootstrap install` MUST appear in the comment so QA2's T7 can pattern-match it. The comment is the audit trail; the manifest is the drift-check baseline.

## Upgrade workflow — request mode (op=upgrade, no approval env)

Request mode is the **first** of two heartbeats that an upgrade spans. It runs all guard checks, computes the diff, and posts a `request_board_approval` — but writes **zero files** under `companies/{id}/`. The full payload-shape contract and env-var protocol are reproduced in [`references/approval-flow.md`](references/approval-flow.md).

1. **Caller identity check (non-negotiable #1).** Identical to install workflow step 1 (resolve `role=ceo` per [`references/role-normalization.md`](references/role-normalization.md), reject with the literal `CEO-only` phrase). T2 of the upgrade Test Gate pattern-matches this.

2. **Target-arg check (non-negotiable #3).** Read the `target` argument. If it is missing, the empty string, the literal string `"latest"`, or any value that does not parse as a tag identifier (a non-empty string with no whitespace), reject with the literal message `"methodology-bootstrap: target argument is required and must be a literal tag (e.g. v1.0.1); received {literal-value}"` and exit before any network call. The literal phrase `target argument is required` MUST appear so QA2's T1 can pattern-match it.

3. **Prior-install check (T3).** Read `companies/{id}/methodology.json`. If the file is missing or unparseable, reject with the literal message `"methodology-bootstrap: no methodology.json at companies/{id}/methodology.json; install first"` and exit before any network call. The literal phrase `install first` MUST appear so QA2's T3 can pattern-match it. Capture `manifest.installedVersion` as `fromVersion` and `manifest.sourceCommitSha` as `fromSha` for the approval payload.

4. **Resolve target tag → 40-char `toSha`.** `GET https://api.github.com/repos/jacobfogg/CopperForge-Method/commits/{target}` (anonymous, `Accept: application/vnd.github+json`, no `Authorization` header). On non-2xx or wrong-length `sha`, reject with the literal `"methodology-bootstrap: target tag {target} did not resolve on jacobfogg/CopperForge-Method; HTTP {status}"` and exit before any disk write. The literal phrase `did not resolve` MUST appear so the existing tag-resolution test surface continues to pattern-match. Rollback (`target` lower than `fromVersion`) is not special-cased here; semver ordering is the board's concern, not the skill's (T8 — same code path).

5. **Fetch both methodology trees + compute file-level diff.** Issue two GitHub tree reads at recursive depth: `GET .../git/trees/{fromSha}?recursive=1` and `GET .../git/trees/{toSha}?recursive=1`. Filter both to `path` starting with `methodology/` and `type=blob`. For every file in the union of both filtered tree sets, fetch only the **target-side** raw content (`GET https://raw.githubusercontent.com/jacobfogg/CopperForge-Method/{toSha}/{path}`); the from-side content is reconstructed from the existing on-disk file plus the manifest's `sha256` (which is authoritative for the from-side). Compute lines-added / lines-removed per file via a standard line-diff against the on-disk current bytes. Build the `diffSummary[]` array per [`references/approval-flow.md`](references/approval-flow.md): one entry per file with `change ∈ {added, modified, removed}` and `linesAdded`/`linesRemoved` (T9 of the upgrade Test Gate verifies accuracy).

6. **Verify the 15-file expected set is present at `toSha`** (same closed list as install workflow step 5; see [`references/source-files.md`](references/source-files.md)). If any of the 15 expected paths is missing at the target SHA, reject with the literal `"methodology-bootstrap: methodology/ tree at {toSha} is missing expected file(s): {comma-separated paths}"` and exit before any approval is posted. The literal phrase `missing expected file` continues to pattern-match this case.

7. **Upload full unified diff as an issue-document artifact (R10).** Concatenate per-file unified diffs (`---`/`+++`/`@@` hunks against the on-disk current bytes vs. the target-side raw content) into a single artifact body. Create the artifact via `POST /api/issues/{originating-issue-id}/documents` with key `methodology-upgrade-diff` (or document-update if a prior rev exists for the same originating issue). Capture the returned canonical URL as `fullDiffArtifactUrl` for the approval payload — the payload itself never inlines the diff (see [`references/approval-flow.md`](references/approval-flow.md) §"Payload size").

8. **POST the approval.** Issue `POST /api/companies/{companyId}/approvals` with `type: "request_board_approval"`, `requestedByAgentId` set to the caller's agent id, `issueIds: [originatingIssueId]`, and the `payload` shape from [`references/approval-flow.md`](references/approval-flow.md): `{ kind: "methodology_upgrade", companyId, fromVersion, toVersion, fromSha, toSha, summary, diffSummary[], fullDiffArtifactUrl, risks[], rollbackVersion: fromVersion }`. Capture the returned approval id. On non-2xx, reject with the literal `"methodology-bootstrap: failed to post approval; HTTP {status}"` and exit before any disk write. The literal phrase `failed to post approval` MUST appear so QA2's T4 can pattern-match the failure case.

9. **Comment + exit.** Post a single comment on the originating issue containing the literal phrase `approval requested, awaiting board` plus the four facts: `fromVersion → toVersion`, `fromSha → toSha`, approval id, and `fullDiffArtifactUrl`. The literal phrase `approval requested, awaiting board` MUST appear so QA2's T4 can pattern-match the success case. Then **exit the heartbeat**. Do NOT poll. Per [`references/approval-flow.md`](references/approval-flow.md), Paperclip wakes this skill in a fresh heartbeat with `PAPERCLIP_APPROVAL_ID` + `PAPERCLIP_APPROVAL_STATUS` + `PAPERCLIP_LINKED_ISSUE_IDS` on resolution. Writing files in request mode is gate-failing (non-negotiable #2).

## Upgrade workflow — apply mode (op=upgrade, approval env present)

Apply mode is the **second** of the two heartbeats that an upgrade spans. It is **only** reachable through the approval-resolution wake. The skill detects apply mode by inspecting the heartbeat env at start; the protocol is reproduced in [`references/approval-flow.md`](references/approval-flow.md).

1. **Apply-mode gate (non-negotiable #2, T5).** Apply mode is entered if and only if all three env vars are present: `PAPERCLIP_APPROVAL_ID` (UUID), `PAPERCLIP_APPROVAL_STATUS` (`approved` or `rejected`), and `PAPERCLIP_LINKED_ISSUE_IDS` (CSV with the originating issue id). If any of the three is missing while `op=upgrade` is asked to apply, reject with the literal `"methodology-bootstrap: apply mode requires approval wake env; missing {comma-separated names}"` and exit before any disk write. The literal phrase `apply mode requires approval wake env` MUST appear so QA2's T5 can pattern-match it.

2. **Status branch.** If `PAPERCLIP_APPROVAL_STATUS == "rejected"`, jump to the **Upgrade rejection path** below (no files change). If `PAPERCLIP_APPROVAL_STATUS == "approved"`, continue with step 3. Any other literal value MUST be rejected with the same `apply mode requires approval wake env` phrase as step 1 — the env var is malformed.

3. **Caller identity re-check (non-negotiable #1, defense-in-depth).** Re-resolve `role=ceo` against the target company's `/agents` response and confirm the caller still matches. The wake protocol carries the originally-requesting agent's id in `requestedByAgentId` of the approval, but a malformed wake or a re-fired wake under a different agent must still be caught. Reject with the literal `CEO-only` phrase exactly as in install workflow step 1.

4. **Read the approval.** `GET /api/approvals/{PAPERCLIP_APPROVAL_ID}`. Capture `payload.toSha`, `payload.toVersion`, `payload.companyId`. If the read fails or the payload's `kind != "methodology_upgrade"`, reject with the literal `"methodology-bootstrap: approval {id} is not a methodology_upgrade; refusing to apply"` and exit before any disk write. This guards against a wrong wake landing in this skill.

5. **Idempotency short-circuit (R5, T-shared).** Read the current `companies/{id}/methodology.json`. If `manifest.sourceCommitSha == payload.toSha`, the upgrade has already been applied (e.g., by an earlier wake whose run interrupted before posting the audit comment, then re-fired). Skip steps 6–7, post the audit-trail comment in step 8 with the literal phrase `already at sourceCommitSha {sha}; no files written`, and exit. The comparison key MUST be `sourceCommitSha`, not `installedVersion`, for the same force-tag reason as install workflow step 8.

6. **Atomic file write (R2, T6).** Apply files via the staging-then-rename pattern documented in [`references/approval-flow.md`](references/approval-flow.md):
   1. Create a per-target-company staging directory `companies/{id}/.methodology-upgrade-{toSha}/` (single ephemeral dir for this run only).
   2. Re-fetch every target-side raw blob (15 files × `raw.githubusercontent.com/.../{toSha}/{path}`), `sha256` each as it lands, and write under the staging dir mirroring the same destination map the install workflow step 6 produces (3 company-root paths + N×per-role paths after the role-fanout).
   3. Per-agent atomic rename: for each `companies/{id}/agents/{agentId}/instructions/`, atomic-rename the staging subtree on top of the live tree (one `rename(2)` per leaf file, or `rename(2)` of the `instructions/` dir against an existing one — whichever the runtime supports atomically). Do company-root files (`COMPANY.md`, `PLATFORM.md`, `CODING_STANDARDS.md`) the same way against `companies/{id}/`.
   4. The staging dir is removed once every rename has succeeded; if any rename fails, the staging dir is left in place for forensic inspection and the manifest is **not** rewritten — the next `drift-check` will surface the partial state.

7. **Manifest write (last).** Update `companies/{id}/methodology.json` with the new `installedVersion`, `installedAt` (wall-clock at apply time), unchanged `repoUrl`, new `sourceCommitSha = payload.toSha`, and a fresh `files[]` array of `{path, sha256}` recomputed from the bytes that were just written. The manifest is written **after** every per-file rename succeeds; partial applies leave the prior manifest untouched (same invariant as install workflow step 9).

8. **Audit-trail comment.** Post a single comment on the originating issue (resolved from `PAPERCLIP_LINKED_ISSUE_IDS`) containing exactly six facts: `fromVersion → toVersion`, `fromSha → toSha`, approval id, file count (`files.length`), apply timestamp, and the literal phrase `role docs updated, re-read at next heartbeat`. The literal phrase `methodology-bootstrap upgrade applied` MUST also appear so QA2 can pattern-match the apply success path. The comment is the audit trail; the updated manifest is the drift-check baseline going forward.

## Upgrade rejection path (op=upgrade, STATUS=rejected)

Reached from apply-mode step 2 when `PAPERCLIP_APPROVAL_STATUS == "rejected"`. T7 of the upgrade Test Gate verifies this path writes zero files.

1. **Read the approval.** `GET /api/approvals/{PAPERCLIP_APPROVAL_ID}` to capture the rejection reason (`approval.decisionReason` or equivalent body field, whichever the approvals API exposes — see [`references/approval-flow.md`](references/approval-flow.md)).

2. **Comment + exit.** Post a single comment on the originating issue (resolved from `PAPERCLIP_LINKED_ISSUE_IDS`) containing the literal phrase `methodology-bootstrap upgrade rejected` plus the rejection reason verbatim and the original `fromVersion → toVersion` so the audit trail is intact. Do **not** touch any file under `companies/{id}/`. Exit the heartbeat.

## Quality bar

- Every reject path uses one of the literal pattern phrases enumerated above (`CEO-only`, `version argument is required`, `did not resolve`, `missing expected file`, `resolved to zero agents`, `matches multiple methodology source dirs`, `failed to fetch`). QA2 pattern-matches these in T1–T4 and T9–T10; rephrasing collapses the test surface.
- All four guard checks (caller identity, version arg, tag resolution, file-set completeness) MUST run before any byte is written to disk. A reject in any of T1–T4 or T9–T10 that leaves a partial write under `companies/{id}/` is gate-failing.
- Company-root files (`COMPANY.md`, `PLATFORM.md`, `CODING_STANDARDS.md`) are written exactly once at `companies/{id}/`, NOT replicated into agent dirs. Per-role files are fanned out to every matching agent. Mixing the two patterns (replicating company-root, or single-copying per-role) is gate-failing on T5.
- The manifest is written last, after every per-file write succeeds. Partial installs leave no manifest. T6 verifies schema; the schema is the closed list `installedVersion | installedAt | repoUrl | sourceCommitSha | files`. Extra fields are gate-failing; missing fields are gate-failing.
- Idempotency (T8) compares `sourceCommitSha`, not `installedVersion`. The justification is in step 8 — force-tag re-pointing must trigger re-write.
- The audit-trail comment (T7) carries exactly the four facts named in step 10. Free-text padding is allowed; omission of any of the four facts is gate-failing.
- Anonymous GitHub access only. Adding an `Authorization` header on any of the three GitHub call sites (`/commits/{tag}`, `/git/trees/{sha}`, `raw.githubusercontent.com/.../{sha}/...`) is gate-failing in v1 — auth is a v1.1 follow-up under the rate-limit response surfaced in [COPAAA-63](/COPAAA/issues/COPAAA-63).
- Writes outside `companies/{id}/` are gate-failing. The skill does NOT touch agent records via PATCH, does NOT modify the company record, does NOT touch other companies' directories. The single CEO-authorized exception is `companies/{id}/methodology.json` itself, which sits at the company root rather than under `agents/`.
- The role-enum normalization table is loaded from [`references/role-normalization.md`](references/role-normalization.md) — a single point of update. Inline-duplicating the table inside the skill body or in a fresh file under any other name is gate-failing on the next maintenance pass; the table belongs in one place so future Paperclip-core role-enum changes need exactly one edit.
- Per-skill README row update is mandatory at Code Gate per the [COPAAA-19](/COPAAA/issues/COPAAA-19) B4 rule. The skill's row in `README.md` accumulates supported operations as each slice lands per Plan §4 C4. With this slice, the row reads `install, upgrade`; the next slice ([COPAAA-79](/COPAAA/issues/COPAAA-79)) accumulates `drift-check`.
- Every reject path in the upgrade workflows uses one of the literal pattern phrases enumerated above (`CEO-only`, `target argument is required`, `install first`, `did not resolve`, `missing expected file`, `failed to post approval`, `apply mode requires approval wake env`). Rephrasing collapses the upgrade Test Gate surface (T1–T5) the same way it would collapse install's.
- **Non-negotiable #2 (board-approval gate).** Request mode of the upgrade operation MUST exit the heartbeat without writing a single byte under `companies/{id}/` (other than the issue-document attachment for the unified diff, which lives on the issue, not under the target company's instance dir). Apply mode MUST be reachable only via all three approval-resolution env vars (`PAPERCLIP_APPROVAL_ID`, `PAPERCLIP_APPROVAL_STATUS`, `PAPERCLIP_LINKED_ISSUE_IDS`); writing files in any other path is gate-failing.
- **Atomic apply (R2, T6).** Apply mode MUST stage every target-side blob under `companies/{id}/.methodology-upgrade-{toSha}/` first, atomic-rename onto the live tree, then write the manifest last. A partial apply (any rename fails) leaves the prior manifest untouched and the staging dir intact for `drift-check` to surface. Writing the manifest before the last rename succeeds is gate-failing.
- **Idempotency on apply (R5).** The apply mode short-circuit comparison MUST be `manifest.sourceCommitSha == payload.toSha`, not `installedVersion`, for the same force-tag reason as install workflow step 8. Apply paths that re-write files when already at the target SHA are gate-failing.
- **Diff accuracy (T9).** The `diffSummary[]` array in the approval payload MUST record one entry per changed file with accurate `linesAdded` / `linesRemoved` integers measured against the on-disk current bytes (not against `fromSha` raw — the on-disk bytes are the source of truth for what the upgrade is actually about to overwrite). Misreporting a diff is gate-failing because the board approves the **diff**, not the version pair alone.
- **Rollback is the same code path (T8).** Rolling back is invoked as `op=upgrade target=<lower version>`. The skill MUST NOT special-case rollback; semver ordering is the board's concern, not the skill's. A rollback-specific reject path is gate-failing.
- **Approval payload shape (BA §3.3, R10).** The payload's `kind` MUST be the literal string `methodology_upgrade`. The full unified diff MUST live behind `fullDiffArtifactUrl` rather than inline. The `rollbackVersion` field MUST equal the manifest's `installedVersion` at the time of request (i.e., `fromVersion`). Deviations from the closed shape in [`references/approval-flow.md`](references/approval-flow.md) are gate-failing.
