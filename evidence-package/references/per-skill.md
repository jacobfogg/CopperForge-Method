# Per-Skill Code-Gate Evidence — Rationale and Worked Examples

This reference expands `SKILL.md` for the per-skill Code Gate path. The contract is canonical at [COPAAA-22 → code-gate-evidence-shape](/COPAAA/issues/COPAAA-22#document-code-gate-evidence-shape); this file gives Dev the rationale, the per-artifact Auditor-challenge surfaces, the out-of-scope list, and two worked examples.

## 1. Why four artifacts and not a one-paragraph summary

Versioning-policy §4.5 originally required only "a one-sentence justification mapped to §4.2." The Plan Gate flagged that this is not mechanically auditable: an Auditor would have to re-read the SKILL.md diff and reconstruct §4.1's mapping by hand to challenge a proposed bump. F1 from the COPAAA-6 Plan Gate review tightened §4.5 into four artifacts so a challenge can be expressed as "Artifact X says Y, but Artifact Z says W." Without that, every Code Gate produces a free-text gate-pass message that is auditor-unchallengeable except by free-text counter-claim. The four artifacts are the floor, not the ceiling — a gate package MAY add smoke probes, dependency-graph notes, or other context.

## 2. Per-artifact Auditor-challenge surfaces

These are the exact challenge expressions an Auditor uses against each artifact. If you cannot articulate what would falsify your artifact in this shape, the artifact is too vague — rewrite it before submitting.

### Artifact A — Semver bump enum classification

The Auditor MUST be able to express a rejection as: *"Artifact A claims `<bump>` citing §4.1 row '<row text>', but Artifact <B|C|D|the SKILL.md diff> says <observation>."* If they cannot articulate that contradiction in one sentence, the classification stands.

Common malformedness patterns:
- Hedged token (`MINOR-leaning`, `MINOR or PATCH`, `approximately MINOR`). Pick one and defend it; gate-failing on its own.
- `§4.1_ROW` cites the section but no row text. The Auditor must be able to grep the policy for the row.
- `§4.2_BREAKING_CHECK` omitted because "the bump is small." §4.2 applies to every bump; a MINOR or PATCH that does not resolve to "non-breaking under §4.2 because …" cannot stand.
- `MAJOR_INSTRUCTION_LIST` field omitted entirely on a non-MAJOR bump. The literal token `N/A` MUST appear; absence is malformed.

### Artifact B — Frontmatter `description` before/after diff

The Auditor reads the diff, classifies the direction independently, and rejects if their reading disagrees with the package's direction token, OR if the direction token violates the cross-consistency rule below.

Cross-consistency rule (Auditor MUST enforce):
- `NARROWED` ↔ `SEMVER_BUMP=MAJOR`. NARROWED with anything else is gate-failing — a description narrowing changes the LLM's invocation behavior and breaks downstream agents whose triggers no longer match.
- `WIDENED` ↔ `SEMVER_BUMP=MINOR`, paired with §4.1 row "`description` rewritten to widen invocation (more triggers, but no triggers removed)".
- `REWRITTEN_SAME_TRIGGER_SET` ↔ `SEMVER_BUMP=PATCH`, with the `triggers_before / triggers_after` line showing identical sets.
- Empty diff (description unchanged) is compatible with any bump — the bump rationale must come from a different §4.1 row.

Common malformedness patterns:
- Branch name or tag in the file headers instead of a 40-char SHA.
- Unchanged-case fence with no body and no "description unchanged at <head SHA>" sentence below — Auditor cannot distinguish "I checked and it's unchanged" from "I forgot."
- Non-empty diff with no direction token directly below the fence.
- `REWRITTEN_SAME_TRIGGER_SET` without the trigger-set comparison line.

### Artifact C — `references/` rename-or-removal grep

The Auditor verifies the commands were actually run (output is literal, not paraphrased) and that ref SHAs match Artifact B. A non-empty `git grep` result (SKILL.md still cites a removed/renamed reference path) is gate-failing as a broken-reference bug, regardless of `SEMVER_BUMP` — the gate fails on the bug, not on the version bump. §4.1's "Removing or renaming a `references/` file SKILL.md still points at" row would also force MAJOR, but only if the bug is fixed first; otherwise the gate fails before bump classification matters.

Common malformedness patterns:
- First command skipped because "no `references/` files moved" — empty output IS the evidence and MUST be reproduced.
- Second command's output paraphrased ("I checked and it's fine") instead of captured verbatim.
- SHAs in commands don't match Artifact B's SHAs (mismatch is gate-failing on its own).

### Artifact D — CHANGELOG.md entry fragment

The Auditor verifies (a) the fragment is paste-ready (no remaining `<…>` placeholders other than the four cited fields, all of which MUST be filled in); (b) the four cross-references — `<skill-key>`, `<SEMVER_BUMP>`, `<gate-issue-identifier>`, §4.1 row text — match Artifacts A, B, C and the gate-review document elsewhere.

Common malformedness patterns:
- Linking to the parent epic ([COPAAA-1](/COPAAA/issues/COPAAA-1)) instead of the per-skill Code Gate issue.
- `<skill-key>` written as the human-readable name (e.g., `Evidence Package`) instead of the canonical slug (`evidence-package`).
- §4.1 row text in the bump-rationale bullet differs from Artifact A's `§4.1_ROW` (verbatim match required).
- Placeholders left in (`<TBD>`, `XXX`, "<one-line summary>"). If the maintainer cannot phrase the customer-observable summary yet, the gate is not ready.

## 3. Where this evidence lives — formatting rules

All four artifacts go inside the gate's `gate-review` document, under a level-2 heading EXACTLY:

```
## Evidence (versioning-policy §4.5 / COPAAA-22)
```

Artifacts appear in order A → D under that heading, each with a level-3 sub-heading naming the artifact. If a gate covers multiple skills, each skill gets its own complete A–D set under its own level-3 heading naming the skill key. Artifacts MUST NOT be shared across skills even when "the change was the same" — the §4.1 row, frontmatter diff, and CHANGELOG fragment are per-skill artifacts.

## 4. Out of scope

This skill defines the per-skill Code-Gate evidence floor. It does NOT cover:

- **Test-Gate evidence** — Test Gates run before Code Gates; their evidence shape is owned by QA, not by this skill.
- **Smoke-test evidence at per-skill gates** — covered by [smoke-test-harness](/COPAAA/issues/COPAAA-7#document-smoke-test-harness); per-skill gates do NOT need smoke-harness output.
- **Doc-reconciliation evidence** — captured at epic close, not at per-skill Code Gates.
- **Branch/tag protection at non-publish gates** — only the v1.0.0 epic-close gate adds that artifact (see `references/epic-close.md`).

A per-skill Code Gate package MAY include other artifacts beyond A–D (additional smoke probes, dependency-graph notes, etc.). This skill is what the Auditor MUST require; the gate may exceed it.

## 5. Worked example — single-skill MINOR bump (description widened, new references file added)

Suppose Dev added a `references/changelog-template.md` to the `auditor-challenge` skill and widened its `description` to mention the new file. Base SHA `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`, head SHA `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`, gate at COPAAA-30.

```
## Evidence (versioning-policy §4.5 / COPAAA-22)

### Artifact A — Semver bump enum classification

SEMVER_BUMP: MINOR
§4.1_ROW: "New `references/` file added (without changing existing-file contracts)"
§4.2_BREAKING_CHECK: non-breaking under §4.2 because no existing reference path was removed or renamed, the description was widened (not narrowed), and no SKILL.md instruction was inverted or removed.
MAJOR_INSTRUCTION_LIST: N/A

### Artifact B — Frontmatter `description` before/after diff

```
--- a/auditor-challenge/SKILL.md (frontmatter `description`, base ref: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa)
+++ b/auditor-challenge/SKILL.md (frontmatter `description`, head ref: bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb)
@@ description @@
-... existing trigger phrases ...
+... existing trigger phrases ... Use this skill when drafting a CHANGELOG entry from a gate review.
```
WIDENED

### Artifact C — `references/` rename-or-removal grep

```
$ git diff --name-status aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb -- 'auditor-challenge/references/'
A       auditor-challenge/references/changelog-template.md

no rename/delete in references/, second command not run
```

### Artifact D — CHANGELOG.md entry fragment

```markdown
### `auditor-challenge` MINOR — Add a CHANGELOG-fragment template for gate reviewers.

- New `references/changelog-template.md` provides a paste-ready CHANGELOG fragment for any reviewer who needs to seed a release note from a single gate.
- Code Gate: [COPAAA-30](/COPAAA/issues/COPAAA-30)
- Bump rationale: §4.1 row "New `references/` file added (without changing existing-file contracts)"
```
```

## 6. Worked example — single-skill PATCH bump (typo fix, no description change)

Same skill, base SHA `cccccccccccccccccccccccccccccccccccccccc`, head SHA `dddddddddddddddddddddddddddddddddddddddd`, gate at COPAAA-31.

```
## Evidence (versioning-policy §4.5 / COPAAA-22)

### Artifact A — Semver bump enum classification

SEMVER_BUMP: PATCH
§4.1_ROW: "Typo fixes"
§4.2_BREAKING_CHECK: non-breaking under §4.2 because no invocation surface, instruction, reference path, slug, or `name` value changed; only spelling.
MAJOR_INSTRUCTION_LIST: N/A

### Artifact B — Frontmatter `description` before/after diff

```
--- a/auditor-challenge/SKILL.md (frontmatter `description`, base ref: cccccccccccccccccccccccccccccccccccccccc)
+++ b/auditor-challenge/SKILL.md (frontmatter `description`, head ref: dddddddddddddddddddddddddddddddddddddddd)
```
description unchanged at dddddddddddddddddddddddddddddddddddddddd

### Artifact C — `references/` rename-or-removal grep

```
$ git diff --name-status cccccccccccccccccccccccccccccccccccccccc dddddddddddddddddddddddddddddddddddddddd -- 'auditor-challenge/references/'

no rename/delete in references/, second command not run
```

### Artifact D — CHANGELOG.md entry fragment

```markdown
### `auditor-challenge` PATCH — Fix two typos in the workflow body.

- Spelling-only fix; no behavior or invocation surface changed.
- Code Gate: [COPAAA-31](/COPAAA/issues/COPAAA-31)
- Bump rationale: §4.1 row "Typo fixes"
```
```

## 7. Multi-skill gates

When a single Code Gate covers more than one skill (rare; usually each Code Gate is per-skill), each skill gets its own complete A–D set under its own level-3 heading naming the skill key. Heading example:

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

#### Artifact A — Semver bump enum classification
…
```

Do not collapse "the change was the same" into one set; the §4.1 row, frontmatter diff, and CHANGELOG fragment are per-skill.

## 8. The first-Code-Gate hand-validation cross-check

Per [smoke-test-harness §5.1](/COPAAA/issues/COPAAA-7#document-smoke-test-harness), the FIRST per-skill Code Gate of the epic — [COPAAA-12](/COPAAA/issues/COPAAA-12), this skill itself — has a chicken-and-egg problem: a defective `evidence-package` skill could in principle produce a passing-looking package describing a failing change. The methodology breaks the chicken-and-egg by requiring the Auditor at COPAAA-12's Code Gate to manually inspect the evidence package against the underlying inputs (test results, diff, doc-update reference) and sign off. Subsequent skills' Code Gates rely on the verified anchor without re-doing the cross-check.

Practical Dev guidance: when assembling the COPAAA-12 evidence package, leave the underlying inputs (the actual `git diff` output, the QA T1–T7 result file, any current-state-doc-update commits) attached to the gate-review document or referenced as comment links so the Auditor can match each artifact to its source without re-running commands.
