---
name: methodology-bootstrap
description: >
  Install a pinned CopperForge methodology release (v1.0.0+ tag of
  jacobfogg/CopperForge-Method) into a target Paperclip company by fetching
  the methodology/ tree at the resolved commit SHA, fanning out per-role files
  to every matching agent's instructions/ directory, writing company-root files
  once at companies/{id}/, and recording the install state in
  companies/{id}/methodology.json. Use this skill when the CEO of a target
  Paperclip company needs to deploy or re-deploy a tagged methodology release.
  This v1 ships only the `install` operation; `upgrade` (with board approval
  gate) and `drift-check` land in COPAAA-69 and COPAAA-79.
---

# methodology-bootstrap (install)

Use this skill when you are the **CEO of a target Paperclip company** and need to install a pinned tagged release of the CopperForge methodology repo (`jacobfogg/CopperForge-Method`) into that company. The install fetches the `methodology/` tree at a 40-char commit SHA the version tag resolved to, applies the role-enum normalization table from [`references/role-normalization.md`](references/role-normalization.md) to fan out per-role files to every matching agent's `instructions/` directory, writes the three company-root files once at `companies/{id}/`, writes the `methodology.json` manifest per [`references/manifest-schema.md`](references/manifest-schema.md), and posts an audit-trail comment on the originating issue. Re-running at the same version is a safe no-op (manifest comparison short-circuits writes). The `upgrade` operation (board-approval gated) lands with [COPAAA-69](/COPAAA/issues/COPAAA-69) and the read-only `drift-check` operation lands with [COPAAA-79](/COPAAA/issues/COPAAA-79).

## Preconditions

- The caller is the CEO of the target company. CEO identity is resolved by the role-enum normalization table in [`references/role-normalization.md`](references/role-normalization.md) (`role=ceo` rule) against the live `GET /api/companies/{id}/agents` response. Any other caller MUST be rejected before any disk read or network call. This is **non-negotiable #1 (CEO-only enforcement)** from the COPAAA-58 plan and is enforced at every operation, not just install.
- The `version` argument is present and is a literal tag identifier (e.g., `v1.0.0`). The string `"latest"`, the empty string, and a missing `version` argument MUST all be rejected before any network call. This is **non-negotiable #3 (version pinning)** from the COPAAA-58 plan; install never resolves "current" or "head" implicitly.
- The canonical methodology repo `https://github.com/jacobfogg/CopperForge-Method` is reachable anonymously over HTTPS. v1 issues no `Authorization` header on any GitHub call; the repo is public and the GitHub anonymous rate limit (60 req/h) covers the 17 reads a single install issues (1 commits, 1 tree, 15 raw blobs).
- The target company id is known and is the company the skill writes into. Writes to disk happen under `companies/{id}/` only; the skill never touches agent records, company records, or any path outside the `companies/{id}/` subtree (with the single exception of `companies/{id}/methodology.json`, which the CEO has explicitly authorized as outside-`agents/` per Q3 of the Plan-Gate ratification).
- The role-enum normalization table in [`references/role-normalization.md`](references/role-normalization.md) is loaded. The two hard-fail rules in that table (`zero-match` and `multi-match`) MUST be enforced at install time before any disk write.

## Workflow

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
- Per-skill README row update is mandatory at Code Gate per the [COPAAA-19](/COPAAA/issues/COPAAA-19) B4 rule. This slice's row records `install` as the only supported operation; B2 and B3 accumulate `upgrade` and `drift-check` onto the same row per Plan §4 C4.
