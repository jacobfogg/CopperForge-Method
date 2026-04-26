---
name: methodology-bootstrap
description: >
  Install a pinned CopperForge methodology release (v1.0.0+ tag of
  jacobfogg/CopperForge-Method) into a target Paperclip company, or run
  a read-only drift-check that compares the company's on-disk methodology
  files against the install manifest. The `install` operation fetches the
  methodology/ tree at the resolved 40-char commit SHA, fans out per-role
  files to every matching agent's instructions/ directory, writes
  company-root files once at companies/{id}/, and records install state in
  companies/{id}/methodology.json. The `drift-check` operation re-hashes
  every manifest entry against on-disk content (no GitHub network calls,
  no writes), flags `modified` / `deleted` / `unexpected` files, and posts
  either a quiet no-drift comment or a structured drift report plus a
  follow-up issue with two recommended responses (revert-to-canonical or
  formalize as methodology-repo PR). Use this skill when the CEO of a
  target Paperclip company needs to deploy a tagged methodology release
  or audit on-disk drift against the install baseline. v1 ships `install`
  and `drift-check`; the `upgrade` operation lands in parallel via
  COPAAA-69.
---

# methodology-bootstrap (install, drift-check)

Use this skill when you are the **CEO of a target Paperclip company** and need to install a pinned tagged release of the CopperForge methodology repo (`jacobfogg/CopperForge-Method`) into that company **or** run a read-only audit of the company's on-disk methodology files against the install manifest. Both operations share the same caller-identity check (`role=ceo` resolved against the target company's `/agents` response per [`references/role-normalization.md`](references/role-normalization.md)) and the same `companies/{id}/methodology.json` schema (see [`references/manifest-schema.md`](references/manifest-schema.md)). They diverge in their effect: `install` fetches over anonymous GitHub and writes per-role + company-root files; `drift-check` performs **zero** GitHub network calls and **zero** disk writes — it re-hashes every manifest entry against on-disk content, scans instrumented locations for unexpected files, and reports the result via [`references/drift-output.md`](references/drift-output.md). The `upgrade` operation (board-approval gated) lands in parallel via [COPAAA-69](/COPAAA/issues/COPAAA-69) on the merged integration commit.

## Preconditions

- The caller passes an `op` argument with the value `install` or `drift-check`. Any other value (including missing or `"upgrade"` until [COPAAA-69](/COPAAA/issues/COPAAA-69) merges) MUST be rejected before any further work. The two operations share preconditions but have separate workflows below.
- The caller is the CEO of the target company. CEO identity is resolved by the role-enum normalization table in [`references/role-normalization.md`](references/role-normalization.md) (`role=ceo` rule) against the live `GET /api/companies/{id}/agents` response. Any other caller MUST be rejected before any disk read or network call. This is **non-negotiable #1 (CEO-only enforcement)** from the COPAAA-58 plan and is enforced at every operation, not just install. Drift-check enforces this same check before reading the manifest, not after, so a non-CEO caller cannot infer manifest contents by reading reject messages.
- The canonical methodology repo `https://github.com/jacobfogg/CopperForge-Method` is reachable anonymously over HTTPS for the `install` operation. v1 issues no `Authorization` header on any GitHub call. The `drift-check` operation does NOT depend on this precondition because it issues no GitHub calls.
- The target company id is known and is the company the skill writes into (or, for `drift-check`, reads from). Writes to disk happen under `companies/{id}/` only; the skill never touches agent records, company records, or any path outside the `companies/{id}/` subtree (with the single exception of `companies/{id}/methodology.json`, which the CEO has explicitly authorized as outside-`agents/` per Q3 of the Plan-Gate ratification). Drift-check writes **nothing** under `companies/{id}/` — read-only is the operation's defining invariant (T1).
- The role-enum normalization table in [`references/role-normalization.md`](references/role-normalization.md) is loaded. The two hard-fail rules in that table (`zero-match` and `multi-match`) MUST be enforced at install time before any disk write. Drift-check does not consume the normalization table at run time — the install manifest already encodes the destination map per agent.
- For `op=drift-check` only: a prior install manifest at `companies/{id}/methodology.json` MUST already exist. Drift-check is not bootstrap; "install first" is a hard precondition (T3 of the drift-check Test Gate).

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
   The `files[]` array MUST contain exactly 15 entries when the install fans out to N agents covering all 6 role-dirs (CEO + CTO + BA + Auditor + QAManager + QA + Dev = 7 source dirs, but the manifest counts destinations not source paths, so the array length is `3 + sum_over_role_dirs(matching_agent_count_for_each_per_role_file)`). The manifest is the single source of truth for `drift-check`; write it last so a partial install (steps 1–7 succeed, step 9 fails) leaves no manifest and `drift-check` will continue to read the prior install's state.

10. **Post the audit-trail comment.** Post a single comment on the originating issue (the issue from which the install was invoked, identified by `PAPERCLIP_LINKED_ISSUE_IDS` or equivalent invocation context) containing exactly four facts: `installedVersion`, `installedAt`, file count (`files.length` from the manifest), and `sourceCommitSha`. The literal phrase `methodology-bootstrap install` MUST appear in the comment so QA2's T7 can pattern-match it. The comment is the audit trail; the manifest is the drift-check baseline.

## Drift-check workflow (op=drift-check)

Drift-check is **read-only** end to end. The operation issues **zero** GitHub network calls (no `commits/{tag}`, no tree, no raw blob reads) and writes **zero** bytes under `companies/{id}/` — including no rewrite of `methodology.json`. The full report-comment + follow-up-issue contract is reproduced in [`references/drift-output.md`](references/drift-output.md) so a maintainer reading the SKILL.md alone can verify the read-only invariant and a maintainer reading the reference file alone has the full output schema.

1. **Caller identity check (non-negotiable #1, T2).** Read the caller's agent identifier from the invocation context (`PAPERCLIP_AGENT_ID` or equivalent). Resolve the target company's CEO by `GET /api/companies/{id}/agents` and applying the `role=ceo` rule from [`references/role-normalization.md`](references/role-normalization.md). If the caller's id does not match the resolved CEO's id, reject with the literal message `"methodology-bootstrap: CEO-only operation; caller {agent-id} is not the CEO of company {company-id}"` and exit **before reading the manifest**. The literal phrase `CEO-only` MUST appear so QA2's T2 can pattern-match it. Reading the manifest only after the CEO check passes prevents a non-CEO caller from inferring manifest contents through error-message side channels.

2. **Manifest-exists check (T3).** Read `companies/{id}/methodology.json`. If the file is missing, unparseable, or fails the closed-shape validation in [`references/manifest-schema.md`](references/manifest-schema.md), reject with the literal message `"methodology-bootstrap: no methodology.json at companies/{id}/methodology.json; install first"` and exit. The literal phrase `install first` MUST appear so QA2's T3 can pattern-match it. Capture `manifest.installedVersion`, `manifest.sourceCommitSha`, and `manifest.files[]` for the comparison loop in step 3.

3. **Compare every manifest entry to on-disk content.** For every `{path, sha256}` entry in `manifest.files[]`:
   1. Resolve the on-disk path as `companies/{id}/{path}`.
   2. If the file is missing, append `{change: "deleted", path, manifestSha256: {entry.sha256}, onDiskSha256: null}` to the drift accumulator.
   3. Otherwise, hash the on-disk bytes (sha256, hex, lowercase). If the digest equals `entry.sha256`, the file is unchanged — do **not** append. If the digest differs, append `{change: "modified", path, manifestSha256: {entry.sha256}, onDiskSha256: {digest}}`.
   4. Drift-check MUST NOT open a file in any mode other than read; it MUST NOT call `mkdir`, `unlink`, `rename`, or any other filesystem-mutating syscall on any path under `companies/{id}/`. Violating this invariant is gate-failing on T1.

4. **Scan instrumented locations for unexpected files.** Build the set of instrumented destination directories from `manifest.files[]`:
   - The company-root: directly under `companies/{id}/`, the three slot names `COMPANY.md`, `PLATFORM.md`, `CODING_STANDARDS.md` are instrumented. Any **other** file directly under `companies/{id}/` is **not** instrumented and MUST NOT be reported (e.g., `methodology.json` itself, ad-hoc operator notes).
   - Per-agent: the set of `agents/{agentId}/instructions/` directory paths derived from `manifest.files[]` entries whose `path` starts with `agents/`. For each such directory, list every file. Any file path in that directory that is NOT in `manifest.files[]` is unexpected — append `{change: "unexpected", path, manifestSha256: null, onDiskSha256: {digest}}`. Note that an unexpected company-root file (a fourth `.md` in `companies/{id}/` with one of the three slot names already in the manifest cannot occur by definition; a wholly different filename in `companies/{id}/` is **not** instrumented and is ignored).
   - The scan is read-only: `readdir` and `stat` only.

5. **No-drift branch (T4).** If the drift accumulator is empty after steps 3 + 4, post **one** quiet success comment on the originating issue (resolved from `PAPERCLIP_LINKED_ISSUE_IDS` or equivalent invocation context) using the no-drift template in [`references/drift-output.md`](references/drift-output.md) §"No-drift comment". The literal phrase `methodology-bootstrap drift-check: no drift detected` MUST appear so QA2's T4 can pattern-match it. Do **not** create a follow-up issue. Do **not** post a board-ping approval. Exit the heartbeat. T4 is the "no green pings" rule from [COPAAA-58 plan §3 B3](/COPAAA/issues/COPAAA-58#document-plan).

6. **Drift branch (T5/T6/T7).** If the drift accumulator is non-empty, do all three of the following in this order; the operation has been read-only up to this point and remains read-only with respect to `companies/{id}/`:
   1. **Post the drift report comment** on the originating issue using the drift-report template in [`references/drift-output.md`](references/drift-output.md) §"Drift-report comment". The literal phrase `methodology-bootstrap drift-check: drift detected` MUST appear. The comment table includes one row per drift entry with columns `change`, `path`, `manifest sha256`, `on-disk sha256`.
   2. **Create the follow-up issue** in the target company via `POST /api/companies/{companyId}/issues` using the follow-up template in [`references/drift-output.md`](references/drift-output.md) §"Follow-up issue". Title: `"methodology-bootstrap drift detected on {companyId} ({totalDriftEntries} entr{y/ies})"`. Description carries the same drift table plus the two recommended responses verbatim ((a) revert to canonical re-install at `manifest.installedVersion`; (b) formalize the drift as a methodology-repo PR). Assignee: the resolved CEO. Priority: `medium`. The phrase `methodology-bootstrap drift detected` MUST appear in both the title and the description.
   3. **Audit-trail tie-back.** The drift-report comment MUST link to the follow-up issue's `identifier`, and the follow-up issue's description MUST link back to the originating issue. Both links use the company-prefixed `/<prefix>/issues/<identifier>` form. The audit trail is the comment + the follow-up issue + the unchanged manifest; until [COPAAA-61](/COPAAA/issues/COPAAA-61) ships the reporter, drift goes only to the originating issue thread per [COPAAA-58 plan §0](/COPAAA/issues/COPAAA-58#document-plan) R8.

7. **Version-vs-drift discrimination (T8).** Drift-check MUST NOT issue any GitHub call to compare the current `default_branch` HEAD against `manifest.sourceCommitSha`. The operation reports "drifted from `manifest.sourceCommitSha`" only and explicitly does NOT report "newer methodology version available." Surfacing a "newer version" suggestion is gate-failing on T8 because that is the `upgrade` operation's concern (see [COPAAA-69](/COPAAA/issues/COPAAA-69)), not drift-check's, and conflating the two scopes encourages operators to re-install (overwriting in-place edits) instead of opening a methodology-repo PR.

## Quality bar

- Every reject path uses one of the literal pattern phrases enumerated above (`CEO-only`, `version argument is required`, `did not resolve`, `missing expected file`, `resolved to zero agents`, `matches multiple methodology source dirs`, `failed to fetch`, `install first`). QA2 pattern-matches these in T1–T4 and T9–T10 of install and T2–T3 of drift-check; rephrasing collapses the test surface.
- All four install guard checks (caller identity, version arg, tag resolution, file-set completeness) MUST run before any byte is written to disk. A reject in any of T1–T4 or T9–T10 that leaves a partial write under `companies/{id}/` is gate-failing.
- Company-root files (`COMPANY.md`, `PLATFORM.md`, `CODING_STANDARDS.md`) are written exactly once at `companies/{id}/`, NOT replicated into agent dirs. Per-role files are fanned out to every matching agent. Mixing the two patterns (replicating company-root, or single-copying per-role) is gate-failing on T5.
- The manifest is written last, after every per-file write succeeds. Partial installs leave no manifest. T6 verifies schema; the schema is the closed list `installedVersion | installedAt | repoUrl | sourceCommitSha | files`. Extra fields are gate-failing; missing fields are gate-failing.
- Idempotency (install T8) compares `sourceCommitSha`, not `installedVersion`. The justification is in install step 8 — force-tag re-pointing must trigger re-write.
- The audit-trail comment (install T7) carries exactly the four facts named in install step 10. Free-text padding is allowed; omission of any of the four facts is gate-failing.
- Anonymous GitHub access only on install. Adding an `Authorization` header on any of the three GitHub call sites (`/commits/{tag}`, `/git/trees/{sha}`, `raw.githubusercontent.com/.../{sha}/...`) is gate-failing in v1 — auth is a v1.1 follow-up under the rate-limit response surfaced in [COPAAA-63](/COPAAA/issues/COPAAA-63).
- **Drift-check is read-only (T1).** No file under `companies/{id}/` may be created, modified, removed, renamed, or have its mode changed by any drift-check invocation. `methodology.json` itself is read-only on this operation; drift-check never rewrites the manifest, even to "refresh" the `installedAt` timestamp. Any filesystem-mutating syscall on a path under `companies/{id}/` originating from a drift-check run is gate-failing on T1, regardless of whether the mutation succeeded.
- **Drift-check makes zero GitHub network calls (T8).** No `api.github.com` request, no `raw.githubusercontent.com` request, no DNS lookup of either host, on any drift-check code path. Even an "upgrade-availability" check is forbidden. Drift versus version-availability is two operations, not one.
- **Drift-check on no drift creates no issue and posts one comment (T4).** The "no green pings" rule is non-negotiable: a clean run produces exactly one originating-issue comment carrying the literal `no drift detected` phrase, and zero side effects elsewhere. Creating an issue, posting an approval, paging the board, or PATCHing any other issue on a clean drift-check run is gate-failing.
- **Drift report scope discipline (T8).** The drift-report comment and the follow-up issue MUST NOT recommend "upgrade to a newer version." The two recommended responses are exactly (a) revert-to-canonical (re-install at `manifest.installedVersion`) and (b) formalize-as-PR (open a methodology-repo PR). Adding a third "(c) upgrade to v{X}" option silently extends drift-check beyond its scope and is gate-failing.
- Writes outside `companies/{id}/` are gate-failing for install. The skill does NOT touch agent records via PATCH, does NOT modify the company record, does NOT touch other companies' directories. The single CEO-authorized exception is `companies/{id}/methodology.json` itself, which sits at the company root rather than under `agents/`. Drift-check makes no writes anywhere.
- The role-enum normalization table is loaded from [`references/role-normalization.md`](references/role-normalization.md) — a single point of update. Inline-duplicating the table inside the skill body or in a fresh file under any other name is gate-failing on the next maintenance pass; the table belongs in one place so future Paperclip-core role-enum changes need exactly one edit.
- Per-skill README row update is mandatory at Code Gate per the [COPAAA-19](/COPAAA/issues/COPAAA-19) B4 rule. The skill's row in `README.md` accumulates supported operations as each slice lands per Plan §4 C4. With this slice the row reads `install, drift-check`; the parallel [COPAAA-69](/COPAAA/issues/COPAAA-69) slice accumulates `upgrade` and the merged integration commit produces the final accumulated form `install, upgrade, drift-check`.
