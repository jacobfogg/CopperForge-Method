# Epic-Close Evidence — v1.0.0 Publish Code Gate

This reference covers the additional artifacts the v1.0.0 epic-close Code Gate requires on top of the per-skill Artifacts A–D (which are still required, one set per skill changed in scope). Per-skill Code Gates do NOT need anything in this file.

## When this fires

Only at the v1.0.0 publish gate (the epic-close subtask that pushes the `v1.0.0` tag, e.g. [COPAAA-20](/COPAAA/issues/COPAAA-20) under the COPAAA-1 decomposition). Every interim per-skill Code Gate before that uses only the per-skill artifacts in `per-skill.md`.

## Additional artifact E — Smoke-summary roll-up (per [smoke-test-harness §5](/COPAAA/issues/COPAAA-7#document-smoke-test-harness))

Source policy: smoke-test-harness §5.2 — at epic close, the harness driver runs end-to-end against the test company, producing `harness/runs/<run-id>/summary.json` with the seven per-skill records. The epic-close evidence package consumes that summary and reproduces, for **every** skill in the v1.0.0 release, the following fields under a level-3 sub-heading:

```
skill canonical key:       copperpress/copperforge-skills/<slug>
seed-issue identifier:     SMOKE-<n>
seed-issue link:           [SMOKE-<n>](/<test-company-prefix>/issues/SMOKE-<n>)
fixture-agent run id:      <agent-run-id>
smoketest revision:
  pre-flight:              <git sha or tag>
  post-run:                <git sha or tag>
  consistent:              true | false
pass/fail:                 pass | fail
lock-acquire timestamp:    <iso8601>
lock-release timestamp:    <iso8601>
```

Required form:

```
## Evidence (versioning-policy §4.5 / COPAAA-22)

### Artifact E — Smoke-summary roll-up (smoke-test-harness §5)

#### copperpress/copperforge-skills/evidence-package
seed-issue identifier:     SMOKE-1
…

#### copperpress/copperforge-skills/current-state-doc-update
seed-issue identifier:     SMOKE-2
…

(repeat for all seven skills)
```

Rules:

- The roll-up MUST cover **every** skill in the v1.0.0 release. Missing a skill is gate-failing.
- `skillRevisionConsistent` (= `pre-flight === post-run`) MUST be `true` for every skill. A `false` value means revision drift during the run; the per-skill record fails regardless of artifact result, and the gate fails.
- The roll-up MUST link each smoke-test result to (a) the corresponding per-skill Code Gate issue, (b) the seed-issue in the test company, and (c) the agent run id. Auditor MUST be able to trace any pass claim back to the originating heartbeat from this artifact alone.
- This satisfies the [COPAAA-1](/COPAAA/issues/COPAAA-1) acceptance line *"Each skill registered in a test Paperclip company and executed at least once successfully."* Without it, that acceptance is unmet.

Auditor-challenge surface: *"Artifact E claims smoke-pass for `<skill>` at run `<run-id>`, but the seed-issue `<SMOKE-n>` shows status `<x>` / no agent run / mismatched revision."*

## Additional artifact F — Tag-protection (per [COPAAA-23](/COPAAA/issues/COPAAA-23) and versioning-policy §4.4 #6 / §5.1 #3)

The package MUST include **exactly one** of (F.a) or (F.b). "We didn't get to it" is NOT acceptable; without one of these, the Auditor MUST block the gate.

### F.a — Configured tag-protection (preferred path)

```
$ gh api /repos/copperpress/copperforge-skills/rules/branches/<branch> -H 'Accept: application/vnd.github+json'
<verbatim JSON response showing tag-protection rule is ACTIVE on pattern `v*`,
 captured at the time of v1.0.0 push>
```

OR a screenshot of GitHub branch/tag protection settings showing the same rule. The pattern MUST be `v*` (covers both stable releases like `v1.0.0` and pre-release tags like `v1.0.0-rc1`). A narrower pattern (e.g., `v*.*.*`) is malformed unless paired with the pre-release-tag Policy line in F.b.

### F.b — Enumerated and evidenced upstream blocker (fallback path)

Exactly one of:

- `gh api /repos/copperpress/copperforge-skills` response showing the GitHub plan tier excludes branch/tag protection for the repo's visibility class, with citation to GitHub's published plan-feature matrix (URL).
- Written refusal of Admin permission from the repo owner, attached as an evidence file.
- An equivalent platform-level constraint identified by name and documented per [versioning-policy §5.1 #3 (iii)](/COPAAA/issues/COPAAA-6#document-versioning-policy).

Plus:

- The README's `## Versioning` section MUST carry the matching enforcement-status statement (per versioning-policy §4.4 minimum content #6, sentence (b)): *"Tag-protection is not enforced at the GitHub level due to [enumerated upstream blocker — name the specific category from §5.1 #3 (i)/(ii)/(iii)]. Tags are protected by author discipline only; please file an issue if you observe a force-moved tag."* The blocker named in the README MUST be the same blocker (with the same evidence) cited here.

Auditor-challenge surface: *"Artifact F.b cites blocker '<x>', but the README's enforcement statement names '<y>'"* — mismatch is gate-failing.

### Pre-release-tag pattern coverage

If the doc is edited before v1.0.0 publish, the protection rule pattern MUST be `v*` (not `v*.*.*`). If the pattern is left at `v*.*.*` for any reason (e.g., the policy doc was not re-revised), the README MUST add a Policy line: *"CopperForge does not ship pre-release tags."* AND the v1.0.0-publish Code Gate MUST verify no `v*-*` tag exists in the repo at gate time, attaching the verification command output.

```
$ git tag -l 'v*-*'
<verbatim output, MUST be empty>
```

A non-empty output is gate-failing as a policy violation.

## Required heading structure at epic close

The full epic-close `gate-review` document carries:

```
## Evidence (versioning-policy §4.5 / COPAAA-22)

### `evidence-package`
#### Artifact A — Semver bump enum classification
…
#### Artifact B — Frontmatter `description` before/after diff
…
#### Artifact C — `references/` rename-or-removal grep
…
#### Artifact D — CHANGELOG.md entry fragment
…

### `current-state-doc-update`
… (A–D for this skill)

### `auditor-challenge`
… (A–D)

### `research-summary`
… (A–D)

### `gate-review`
… (A–D)

### `test-change-request`
… (A–D)

### `doc-reconciliation`
… (A–D)

### Artifact E — Smoke-summary roll-up (smoke-test-harness §5)
… (one block per skill, per §5 table above)

### Artifact F — Tag-protection
… (F.a OR F.b, plus pre-release pattern coverage if applicable)
```

Cross-consistency rules at epic close:

- Artifact E covers exactly the seven skills, no more, no less.
- The CHANGELOG.md fragments from each skill's Artifact D, when concatenated under a single `## v1.0.0 — <date>` heading, form the v1.0.0 CHANGELOG entry. The maintainer pastes them in tag-cut order; the gate reviews them as already-paste-ready (no rework allowed at the gate).
- Artifact F's chosen path (F.a or F.b) MUST match the README's `## Versioning` section enforcement-status statement (a) or (b) verbatim on the named blocker.

## Out of scope at epic close

- Doc-reconciliation evidence — owned by the doc-reconciliation skill, captured in a separate artifact under the doc-reconciliation epic-close subtask, not by this skill.
- Any per-skill Code-Gate evidence that was already produced at the per-skill gate — Artifacts A–D for skills changed only at epic-close (e.g., a README edit on the publish commit) still get fresh A–D sets here, but skills whose only change shipped at an earlier per-skill Code Gate are NOT re-emitted; the prior gate's `gate-review` document is referenced by link from Artifact D's `Code Gate:` bullet.

## References

- Smoke-test harness output and §5 integration: [COPAAA-7 → smoke-test-harness](/COPAAA/issues/COPAAA-7#document-smoke-test-harness)
- Tag-protection requirement: [COPAAA-23](/COPAAA/issues/COPAAA-23) (description) + [versioning-policy §4.4 #6, §5.1 #1, §5.1 #3](/COPAAA/issues/COPAAA-6#document-versioning-policy)
- Source contract for Artifacts A–D (still required at epic close): [COPAAA-22 → code-gate-evidence-shape](/COPAAA/issues/COPAAA-22#document-code-gate-evidence-shape)
