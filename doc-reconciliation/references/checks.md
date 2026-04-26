# `doc-reconciliation` — R1–R4 mechanics, Auditor-challenge surfaces, out-of-scope list, and worked examples

This file is load-on-demand reference material for the [`doc-reconciliation`](../SKILL.md) skill. It explains *why* each R-check is shaped the way it is, what an Auditor can legitimately challenge a reconciliation report on, what is explicitly out of scope, and what each verdict (`RECONCILED`, `DRIFT`, `STALE`) looks like with literal artifacts. Read this when you are running the v1.0.0 epic-close reconciliation pass and want to apply the workflow to a real repo state, or when you are the Auditor of record for that pass and want to know what challenge surfaces are in bounds.

## 1. Why R1–R4 instead of "read every doc"

The CopperForge methodology calibrates against an audit-trail-first contract: the Code-Gate evidence package ([versioning-policy §4.5](/COPAAA/issues/COPAAA-6#document-versioning-policy)) and the per-skill Code-Gate artifact set ([COPAAA-22](/COPAAA/issues/COPAAA-22)) are mechanical — every claim is a SHA-stamped string match the Auditor can falsify in one sentence. An epic-close reconciliation pass that asked "do the docs feel right?" would re-introduce free-text gate-pass discretion that the per-skill gates already removed.

R1–R4 are the four mechanical doc surfaces where drift can hide between per-skill gates:

- **R1** is the only place the README's purpose-statement table touches per-skill `description` frontmatter. Per-skill Code Gates check the `description` *of that skill* (Artifact B), but no per-skill gate cross-checks against the README row that markets the skill at the repo root. The QAM rule ratified at [COPAAA-19 → code-gate-review §5](/COPAAA/issues/COPAAA-19#document-code-gate-review) closed the per-skill side of this gap; R1 closes the cross-skill side at epic close.
- **R2** is where the CHANGELOG.md (a single repo-root file) accumulates per-skill Artifact D fragments across the epic. Each per-skill Code Gate confirms its own fragment is paste-ready (per [COPAAA-12 evidence-package SKILL.md step 8](/COPAAA/issues/COPAAA-12)), but the *aggregate* CHANGELOG is the reader-facing surface and is not gated on a per-skill basis.
- **R3** re-runs the per-skill `references/` rename-or-removal grep ([versioning-policy §4.5 / evidence-package Artifact C](/COPAAA/issues/COPAAA-6#document-versioning-policy)) across the whole repo at `epic-head-SHA`. A per-skill Code Gate that landed earlier in the epic could not see citations introduced later by a sibling skill; R3 is the only check that runs after every per-skill commit is on `main`.
- **R4** binds methodology rules ratified mid-epic (the kind of rule [COPAAA-19](/COPAAA/issues/COPAAA-19) §5 introduces, ratifying the per-skill README-row-update at the gate decision itself) to the doc artifacts they require. Mid-epic methodology drift is the highest-friction class of drift: it is invisible in any single skill's diff, and only an epic-scoped pass catches it.

Each R-check is mechanical because each one consumes a literal artifact at `epic-head-SHA` (a README row's `Purpose` cell, a CHANGELOG fragment, a `git ls-tree` lookup, a doc-artifact existence check). No semantic judgment lives in the check itself; the verdict comes from a string compare or a path resolution.

## 2. Auditor-challenge surfaces (what the Auditor MAY legitimately challenge)

The Auditor of record at the v1.0.0 epic-close gate may challenge this skill's reconciliation report on:

- **SHA fidelity.** Any SHA cited in any R-block that is not exactly 40 hex chars, or that does not resolve on `github.com/jacobfogg/CopperForge-Skills`, is a Blocker.
- **Verdict-evidence pairing.** A `RECONCILED` verdict with no `R*_EVIDENCE_PATH` entry, or a `DRIFT`/`STALE` verdict whose evidence path does not actually contain the claimed text, is a Blocker.
- **Hedged outcomes.** "Mostly reconciled," "RECONCILED-with-handwave," or "we'll fix it after the tag" outcomes are Blockers — the workflow §9 enumerates the three legal outcomes (`RECONCILED-CLEAN`, `RECONCILED-WITH-CONDITIONS`, `BLOCKED`), and any other outcome is malformed.
- **Inventory completeness.** If `git ls-tree -r <epic-head-SHA>` shows a top-level `<skill-name>/SKILL.md` that does not appear as an `R1_SKILL` row, that is a Blocker. The pass must enumerate every shipped skill.
- **R4 source provenance.** If an `R4_RULE` is sourced from a comment thread, a research summary, or any document other than a `gate-review` or `code-gate-review` document closed before `epic-head-SHA`, that is a Blocker — the pass exists to verify gate decisions, not to import speculative rules.
- **Fix-forward ticket plausibility.** For a `RECONCILED-WITH-CONDITIONS` outcome, every cited `R*_FOLLOWUP` ticket must (a) exist on the company board, (b) be in `todo`/`in_progress`/`backlog` status (not `done` already and not `cancelled`), and (c) name the same drift the R-block names. Any mismatch is a Blocker.

The Auditor MAY NOT challenge:

- The choice of R1–R4 as the four checks. The rubric is fixed by [research-summary §3.6](/COPAAA/issues/COPAAA-2#document-research-summary) (T1–T7 for SKILL.md structure, R1–R4 for epic-close doc surfaces); auditor challenges to the rubric belong at the human review surface (BA at Plan Gate, peer review at Code Gate of this skill itself), not at the epic-close gate where this skill executes.
- The wording of any per-skill `description` in itself. Per-skill `description` was already gated at the per-skill Code Gate (Artifact B) via [evidence-package SKILL.md step 4](/COPAAA/issues/COPAAA-12). R1 only cross-checks the `description` *against the README row*, not against an a-priori "good description" rubric.
- Skill behavior beyond docs. The reconciliation pass is doc-only. If the Auditor wants to challenge whether a shipped skill *actually does what its description claims*, that belongs to the smoke-test harness ([COPAAA-7](/COPAAA/issues/COPAAA-7) §5) and the test company's published-skill verification, not here.

## 3. Explicitly out of scope (will not affect the verdict)

The reconciliation pass does NOT check:

- **Wording quality of any doc artifact.** "Is this README row's Purpose cell well-written?" is a BA / peer-review concern. R1 only asks "is the cell consistent with the SKILL.md description?"
- **Section ordering inside SKILL.md bodies.** Whether a SKILL.md puts "Quality bar" before or after "Workflow" is governed by the per-skill T1–T7 structural test ([research-summary §3.6](/COPAAA/issues/COPAAA-2#document-research-summary)); it does not affect epic-close reconciliation.
- **`references/` file count, byte cap, or rubric compliance.** [Research-summary §3.4](/COPAAA/issues/COPAAA-2#document-research-summary) is authoring guidance; R3 only checks that *cited* references files exist at `epic-head-SHA`, not that the right files were authored.
- **CHANGELOG.md ordering of entries within a single per-skill section.** R2 checks fragment *presence* and *commit-order* between skills, not within a single skill's CHANGELOG section.
- **Whether the `v1.0.0` tag has been pushed.** Tag-protection and tag-immutability evidence are produced by `evidence-package` references/epic-close.md ([COPAAA-12](/COPAAA/issues/COPAAA-12)) and consumed at the epic-close Code Gate. The reconciliation pass runs *before* the tag is cut and assumes the tag is not yet on the remote.
- **README registration-mechanics correctness.** [COPAAA-19](/COPAAA/issues/COPAAA-19) gated the four-source registration documentation; the reconciliation pass does not re-gate that decision.
- **The `version` line in any frontmatter.** `version` is not a versioning event per [versioning-policy §4.5](/COPAAA/issues/COPAAA-6#document-versioning-policy); the agent-observable surface is `name`, `description`, body, and `references/*.md` paths. Reconciliation tracks those.

## 4. Worked R1 examples

Skill in scope: `evidence-package` at hypothetical `epic-head-SHA = 0123456789abcdef0123456789abcdef01234567`.

### 4.1 R1 — RECONCILED

`README.md:23` (at `epic-head-SHA`):

```
| [`evidence-package`](./evidence-package/) | Bundle code, tests, and gate-relevant artifacts into a single Code-Gate-ready packet. |
```

`evidence-package/SKILL.md` frontmatter `description` (one-line collapse):

```
Assemble the per-skill Code-Gate evidence package — semver-bump enum, frontmatter description before/after diff, references/ rename-or-removal grep, and CHANGELOG fragment — placed under the canonical heading in the gate's review document so the Auditor can challenge any claim against a literal artifact instead of re-reading the diff. Use this skill when preparing the evidence section of a CopperForge per-skill Code Gate, or when assembling the v1.0.0 epic-close Code-Gate evidence package.
```

R1-block:

```
R1_SKILL: evidence-package
R1_README_ROW: "Bundle code, tests, and gate-relevant artifacts into a single Code-Gate-ready packet."
R1_SKILL_DESCRIPTION_HEAD: "Assemble the per-skill Code-Gate evidence package — semver-bump enum, frontmatter description before/after diff, references/ rename-or-removal grep, and CHANGELOG fragment — …"
R1_VERDICT: RECONCILED
R1_EVIDENCE_PATH: README.md:23
```

The substring match is on the canonical purpose noun-phrase ("Code-Gate evidence" / "Code-Gate-ready packet"). The README row is a faithful one-line collapse of the description's purpose-statement; no DRIFT.

### 4.2 R1 — DRIFT (hypothetical)

Suppose during the epic, the `evidence-package` skill's `description` was widened (a MINOR bump) to also cover the v1.0.0 epic-close gate. The README row was not updated. Then:

R1-block:

```
R1_SKILL: evidence-package
R1_README_ROW: "Bundle code, tests, and gate-relevant artifacts into a single Code-Gate-ready packet."
R1_SKILL_DESCRIPTION_HEAD: "Assemble the per-skill Code-Gate evidence package … OR the v1.0.0 epic-close Code-Gate evidence package …"
R1_VERDICT: DRIFT
R1_EVIDENCE_PATH: README.md:23
R1_FOLLOWUP: [COPAAA-99](/COPAAA/issues/COPAAA-99)
```

The README row claims the skill produces a per-skill packet only; the description widened to cover the epic-close packet too. The fix-forward ticket COPAAA-99 must update the README row to mention the epic-close path, OR the description must narrow back to per-skill-only. Either fix is acceptable; the reconciliation pass does not pick.

### 4.3 R1 — STALE (hypothetical)

Suppose a skill was renamed mid-epic from `gate-review` to `gate-record`, with a MAJOR bump per [versioning-policy §4.1](/COPAAA/issues/COPAAA-6#document-versioning-policy). The README row was not updated:

R1-block:

```
R1_SKILL: gate-record
R1_README_ROW: "[`gate-review`](./gate-review/) | Write the canonical Plan / Test / Code / Epic Gate decision document."
R1_SKILL_DESCRIPTION_HEAD: "(no SKILL.md at gate-review/ at epic-head-SHA — directory renamed to gate-record/)"
R1_VERDICT: STALE
R1_EVIDENCE_PATH: README.md:27
R1_FOLLOWUP: [COPAAA-100](/COPAAA/issues/COPAAA-100)
```

The README row points at a directory that no longer exists at `epic-head-SHA`. Reconciliation flags it `STALE`; the fix-forward ticket renames the README row. Note: this scenario would also produce an R3 finding for any other skill that cited `gate-review/references/...` paths.

## 5. Worked R2 examples

### 5.1 R2 — RECONCILED

Suppose the per-skill Code-Gate commits in the epic are (in order):

```
8ec9d77 Add evidence-package skill (COPAAA-12)
abc1234 Add current-state-doc-update skill (COPAAA-13)
def5678 Add auditor-challenge skill (COPAAA-14)
```

`CHANGELOG.md` at `epic-head-SHA` opens with:

```markdown
## v1.0.0 (epic close)

### `evidence-package` MINOR — first per-skill Code-Gate evidence shape …
- …
- Code Gate: [COPAAA-12](/COPAAA/issues/COPAAA-12)
- Bump rationale: §4.1 row "New skill added to the repo (no prior agent-observable surface)"

### `current-state-doc-update` MINOR — …
- Code Gate: [COPAAA-13](/COPAAA/issues/COPAAA-13)

### `auditor-challenge` MINOR — …
- Code Gate: [COPAAA-14](/COPAAA/issues/COPAAA-14)
```

R2-blocks (one per skill):

```
R2_SKILL: evidence-package
R2_CODE_GATE_COMMIT: 8ec9d7758c99d35720888b6afc56cd410627a84a
R2_FRAGMENT_PRESENT: YES
R2_FRAGMENT_BUMP_TOKEN: MINOR
R2_VERDICT: RECONCILED
R2_EVIDENCE_PATH: CHANGELOG.md:3-7
```

(and similarly for the other two). Fragment order in CHANGELOG matches commit order; bump tokens match each per-skill Artifact A.

### 5.2 R2 — DRIFT (missing fragment)

If the `auditor-challenge` per-skill Code Gate's Artifact D was forgotten:

```
R2_SKILL: auditor-challenge
R2_CODE_GATE_COMMIT: def5678…(40-char)
R2_FRAGMENT_PRESENT: NO
R2_FRAGMENT_BUMP_TOKEN: N/A
R2_VERDICT: DRIFT
R2_EVIDENCE_PATH: CHANGELOG.md:1-12 (no fragment with `auditor-challenge` slug)
R2_FOLLOWUP: [COPAAA-101](/COPAAA/issues/COPAAA-101)
```

The fix-forward ticket appends the missing Artifact D fragment to CHANGELOG.md.

## 6. Worked R3 example

### 6.1 R3 — RECONCILED

Pointer line in `evidence-package/SKILL.md` at `epic-head-SHA`:

```
For the rationale behind each rule … read: `skills/evidence-package/references/per-skill.md`

For the additional artifacts the v1.0.0 epic-close Code Gate requires … read: `skills/evidence-package/references/epic-close.md`
```

Note: the canonical pointer form per [research-summary §3.3 step 6](/COPAAA/issues/COPAAA-2#document-research-summary) uses `skills/<skill-name>/references/<topic>.md`; the on-disk path at `epic-head-SHA` is `evidence-package/references/per-skill.md` (the `skills/` prefix is the agent-side install convention, not a repo path). R3 normalizes the pointer by stripping the leading `skills/` segment to derive the repo-relative path.

R3-block:

```
R3_SKILL: evidence-package
R3_CITED_PATHS:
  - evidence-package/references/per-skill.md
  - evidence-package/references/epic-close.md
R3_RESOLVED_AT_HEAD:
  - evidence-package/references/per-skill.md: PRESENT
  - evidence-package/references/epic-close.md: PRESENT
R3_VERDICT: RECONCILED
R3_EVIDENCE_PATH: evidence-package/SKILL.md:75-79
```

### 6.2 R3 — STALE

Suppose `references/per-skill.md` was renamed to `references/per-skill-evidence.md` mid-epic (a MAJOR bump per §4.1 row "Removing or renaming a `references/` file that `SKILL.md` still points at") but `evidence-package/SKILL.md` body was not updated:

```
R3_SKILL: evidence-package
R3_CITED_PATHS:
  - evidence-package/references/per-skill.md
  - evidence-package/references/epic-close.md
R3_RESOLVED_AT_HEAD:
  - evidence-package/references/per-skill.md: MISSING
  - evidence-package/references/epic-close.md: PRESENT
R3_VERDICT: STALE
R3_EVIDENCE_PATH: evidence-package/SKILL.md:75
R3_FOLLOWUP: [COPAAA-102](/COPAAA/issues/COPAAA-102)
```

The fix-forward ticket either restores the old filename or updates the SKILL.md pointer.

## 7. Worked R4 example

The QAM rule ratified at [COPAAA-19 → code-gate-review §5](/COPAAA/issues/COPAAA-19#document-code-gate-review) requires every per-skill Code Gate to include a "README row update" section in its evidence package. The required doc artifact is the row update reflected in `README.md` at the per-skill Code-Gate commit.

R4-block (illustrative, using `evidence-package`):

```
R4_RULE: "Each per-skill Code Gate MUST include in its evidence-package a 'README row update' section."
R4_RATIFIED_AT: [COPAAA-19](/COPAAA/issues/COPAAA-19#document-code-gate-review)
R4_REQUIRED_DOC_ARTIFACT: README.md:23 (purpose-statement row for `evidence-package`, present at epic-head-SHA)
R4_PRESENT: YES
R4_VERDICT: RECONCILED
R4_EVIDENCE_PATH: README.md:23
```

If the per-skill Code Gate for some skill X failed to update the README row (and the row is still empty or stale at `epic-head-SHA`), R4 produces `R4_PRESENT: NO` / `R4_VERDICT: DRIFT` and the epic-close gate cannot pass without a fix-forward ticket.

## 8. Out-of-scope reminder for the Epic-Gate Auditor

The reconciliation pass is doc-surface-only. It does NOT verify that the smoke-test harness ([COPAAA-7](/COPAAA/issues/COPAAA-7)) ran clean against the published skills, that tag protection ([COPAAA-23](/COPAAA/issues/COPAAA-23)) is configured, or that the `v1.0.0` tag has been cut. Those are separate epic-close artifacts produced by `evidence-package` ([COPAAA-12](/COPAAA/issues/COPAAA-12)) under the §4.5 epic-close requirements. Reconciliation runs first; the smoke-test + tag-protection artifacts run after a `RECONCILED-CLEAN` (or `RECONCILED-WITH-CONDITIONS`) outcome.
