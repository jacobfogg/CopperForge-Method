---
name: doc-reconciliation
description: >
  Run the v1.0.0 epic-close reconciliation pass for a CopperForge skill
  release — compare every current-state doc artifact (README purpose-statement
  rows, per-skill SKILL.md frontmatter descriptions, `references/` pointers
  cited from each SKILL.md, CHANGELOG fragments) against the shipped
  epic-close HEAD SHA and produce a four-section drift report (R1–R4) the
  Epic-Gate reviewers can challenge against literal artifacts. Use when
  preparing the doc-reconciliation section of the v1.0.0 epic-close gate, or
  any subsequent epic that closes a multi-skill batch on this repo.
---

# Doc Reconciliation

Use this skill when you are about to open the v1.0.0 epic-close gate (or any later multi-skill epic-close gate) on `jacobfogg/CopperForge-Skills` and must reconcile the doc surface against the shipped code at the epic-close HEAD. It produces four mechanical reconciliation reports (R1–R4) under a fixed heading in the epic-close `gate-review` document, so the Auditor can challenge any drift claim against a literal artifact instead of re-reading the entire repo. The methodology rationale, the per-check Auditor-challenge surfaces, the out-of-scope list, and worked R1–R4 examples are in `references/checks.md`; this body specifies the workflow.

## Preconditions

- Every per-skill Code Gate in the epic has decided PASS or PASS-WITH-CONDITIONS, and the resulting commits are on `main` of `github.com/jacobfogg/CopperForge-Skills`.
- The `evidence-package` skill ([COPAAA-12](/COPAAA/issues/COPAAA-12)) has been used at every per-skill Code Gate; CHANGELOG fragments (Artifact D) are committed in commit order.
- You can resolve both 40-char SHAs for the epic: `epic-base-SHA` (the `main` HEAD immediately before the first per-skill Code-Gate commit) and `epic-head-SHA` (the `main` HEAD immediately before the proposed `v1.0.0` tag push).
- The epic-close gate issue exists and you can create or update its `gate-review` document.
- You have read [research-summary §3.1, §3.2, §3.6](/COPAAA/issues/COPAAA-2#document-research-summary), [versioning-policy §4.4, §4.5](/COPAAA/issues/COPAAA-6#document-versioning-policy), and the per-skill evidence contract at [COPAAA-22 → code-gate-evidence-shape](/COPAAA/issues/COPAAA-22#document-code-gate-evidence-shape). This skill enforces consistency across those contracts; it does not redefine them.

## Workflow

1. **Resolve refs.** Capture both 40-char commit SHAs (`epic-base-SHA`, `epic-head-SHA`). Branch names, tag names, and "current" are malformed. Both SHAs MUST appear verbatim in every R1–R4 artifact below; mismatched SHAs across artifacts is gate-failing on its own.

2. **Enumerate the doc inventory at `epic-head-SHA`.** Run `git ls-tree -r <epic-head-SHA> --name-only` and capture the verbatim subset matching:

   - `README.md` (repo root)
   - `<skill-name>/SKILL.md` (each top-level skill directory)
   - `<skill-name>/references/*.md` (each skill's references/ tree, if present)
   - `CHANGELOG.md` (repo root)

   Empty inventory rows are gate-failing — at v1.0.0 epic close, every shipped skill MUST have a SKILL.md and a CHANGELOG entry.

3. **R1 — README row ↔ SKILL.md description consistency.** For every shipped skill, compare its README purpose-statement row's `Purpose` cell against its `SKILL.md` `description` folded scalar at `epic-head-SHA`. Use the literal block:

   ```
   R1_SKILL: <skill-key>
   R1_README_ROW: "<Purpose cell text, exact>"
   R1_SKILL_DESCRIPTION_HEAD: "<one-line collapse of description folded scalar at epic-head-SHA>"
   R1_VERDICT: RECONCILED | DRIFT | STALE
   R1_EVIDENCE_PATH: README.md:<line-number-at-epic-head-SHA>
   ```

   Verdict semantics: `RECONCILED` = README row is a faithful one-line collapse of the SKILL.md description's purpose-statement (substring match on the canonical purpose noun-phrase). `DRIFT` = the row claims a behavior the description does not. `STALE` = the row references an old skill name, an old slug, or a removed skill. Hedged verdicts ("mostly-reconciled", "DRIFT-leaning") are gate-failing.

4. **R2 — CHANGELOG.md ↔ per-skill Code-Gate Artifact D fragments.** Run `git log --pretty=format:'%H %s' <epic-base-SHA>..<epic-head-SHA> -- '<skill-dir>/'` for every shipped skill, then `git show <epic-head-SHA>:CHANGELOG.md` and capture verbatim. For every per-skill Code-Gate commit identified in the log, locate its corresponding Artifact D fragment in CHANGELOG.md by `<skill-key>` + `<SEMVER_BUMP>` token. Use the literal block:

   ```
   R2_SKILL: <skill-key>
   R2_CODE_GATE_COMMIT: <40-char SHA>
   R2_FRAGMENT_PRESENT: YES | NO
   R2_FRAGMENT_BUMP_TOKEN: MAJOR | MINOR | PATCH | N/A
   R2_VERDICT: RECONCILED | DRIFT
   R2_EVIDENCE_PATH: CHANGELOG.md:<line-range-at-epic-head-SHA>
   ```

   `R2_FRAGMENT_PRESENT: NO` is gate-failing as a missing-CHANGELOG-row drift; `R2_FRAGMENT_BUMP_TOKEN` MUST match the Artifact A token recorded at that per-skill Code Gate. Fragments out of commit order are also `DRIFT` (the CHANGELOG must mirror the commit timeline so post-epic readers can follow the build sequence).

5. **R3 — `references/` pointer ↔ filesystem reality.** For every shipped skill, run `git grep -n -F 'references/' <epic-head-SHA> -- '<skill-dir>/SKILL.md'` and capture the verbatim output. For each cited path, verify the file exists at `epic-head-SHA` via `git ls-tree <epic-head-SHA> -- '<cited-path>'`. Use the literal block:

   ```
   R3_SKILL: <skill-key>
   R3_CITED_PATHS:
     - <cited path 1>
     - <cited path 2>
   R3_RESOLVED_AT_HEAD:
     - <cited path 1>: PRESENT | MISSING
     - <cited path 2>: PRESENT | MISSING
   R3_VERDICT: RECONCILED | STALE
   R3_EVIDENCE_PATH: <skill-dir>/SKILL.md:<line-numbers-at-epic-head-SHA>
   ```

   Any `MISSING` line is `STALE` and gate-failing as a broken-reference drift, separate from the per-skill bump. This is a re-run of the per-skill `evidence-package` Artifact C across the whole repo at `epic-head-SHA`, on the assumption that an Artifact C run at a per-skill Code Gate may have missed cross-skill citations introduced later in the epic.

6. **R4 — Mid-epic methodology bindings ↔ doc artifacts.** For every methodology rule ratified mid-epic in a `gate-review` or `code-gate-review` document (e.g., "README row update at per-skill Code Gate" from [COPAAA-19](/COPAAA/issues/COPAAA-19#document-code-gate-review) §5), verify the corresponding doc artifact exists at `epic-head-SHA`. Use the literal block:

   ```
   R4_RULE: "<rule one-liner, exact, from the ratifying gate-review document>"
   R4_RATIFIED_AT: [<gate-review issue identifier>](/COPAAA/issues/<identifier>#document-<doc-key>)
   R4_REQUIRED_DOC_ARTIFACT: <file path or pattern at epic-head-SHA>
   R4_PRESENT: YES | NO
   R4_VERDICT: RECONCILED | DRIFT
   R4_EVIDENCE_PATH: <file path>:<line-numbers-at-epic-head-SHA>
   ```

   Source the rule list from the epic's `gate-review` and `code-gate-review` documents only — do not import rules from comment threads or research summaries (those are inputs to the gates, not gate decisions). `R4_PRESENT: NO` is `DRIFT` and gate-failing.

7. **Place artifacts in the epic-close `gate-review` doc.** Inside the epic-close gate's `gate-review` document, create exactly this level-2 heading:

   ```
   ## Doc reconciliation (research-summary §3.6 / versioning-policy §4.4 / COPAAA-18)
   ```

   Under it, add R1 → R4 in order, each with a level-3 sub-heading (`### R1 — README row ↔ SKILL.md description consistency`, `### R2 — CHANGELOG.md ↔ per-skill Code-Gate Artifact D fragments`, `### R3 — references/ pointer ↔ filesystem reality`, `### R4 — Mid-epic methodology bindings ↔ doc artifacts`). Each R-section repeats its literal block once per shipped skill (R1, R2, R3) or once per ratified rule (R4); never collapse multiple skills into a single block.

8. **Cross-consistency self-check.** Before handing the epic-close gate off, verify:
   - Every SHA cited in any R-block is exactly 40 hex chars and identical to `epic-head-SHA` (or `epic-base-SHA` where the workflow names it).
   - Every R1 `R1_SKILL` value appears as a row in `README.md` at `epic-head-SHA` AND as a `<skill-name>/SKILL.md` at `epic-head-SHA`. A skill present in one and absent in the other is itself `DRIFT`.
   - Every R2 `R2_CODE_GATE_COMMIT` SHA is reachable from `epic-head-SHA` (`git merge-base --is-ancestor <commit> <epic-head-SHA>`).
   - Every R3 cited path is a relative path under `<skill-dir>/`; absolute paths or paths escaping the skill directory are `STALE` regardless of presence.
   - Every R4 `R4_RATIFIED_AT` link resolves to a `gate-review` or `code-gate-review` document on a CopperForge issue closed before `epic-head-SHA`.
   Any mismatch is gate-failing; fix forward (file a follow-up per-skill ticket, do not edit the gate-review doc to paper over the drift) before re-submitting.

9. **Decide outcome.** The reconciliation pass concludes with exactly one of:
   - `RECONCILED-CLEAN` — every R1–R4 verdict is `RECONCILED`.
   - `RECONCILED-WITH-CONDITIONS` — at least one `DRIFT` or `STALE` verdict, AND every such finding has a fix-forward ticket cited in the same R-block (`R*_FOLLOWUP: [<identifier>](/COPAAA/issues/<identifier>)`). The epic-close gate may still pass with conditions if the conditions exactly match the followup tickets.
   - `BLOCKED` — at least one `DRIFT` or `STALE` verdict without a fix-forward ticket. The epic-close gate cannot pass; the v1.0.0 tag MUST NOT be cut.
   Hedged outcomes ("mostly-reconciled", "tag-but-add-followup-after") are gate-failing on their own.

## Quality bar

- Every R1–R4 row produces literal evidence (a file path with line numbers, or verbatim git output). "I checked and it's fine" without an `R*_EVIDENCE_PATH` entry is malformed.
- Every SHA cited is exactly 40 hex chars and matches across artifacts.
- The reconciliation pass runs ONCE per epic, against a frozen `epic-head-SHA`. If new commits land on `main` after the pass starts, restart from step 1 with the new HEAD — never patch a half-written reconciliation forward.
- The skill produces evidence; the gate decision belongs to QAM with Auditor challenge. A `BLOCKED` outcome from this skill is a fact about the doc surface, not a request for a methodology relaxation.
- This skill MUST NOT generate fixes. If R1 finds a README-row drift, file a fix-forward ticket and let the per-skill author land the correction; do not edit `README.md` from inside this pass.
- Every claim that survives this check must be Auditor-falsifiable in one sentence: "R-block X claims Y, but the file at `epic-head-SHA` says Z." If you cannot articulate what would falsify each claim, the artifact is too vague — rewrite it.
- For the FIRST v1.0.0 epic-close pass run with this skill: the reconciliation report MUST be hand-validated by the Auditor against the underlying inputs (README rows, SKILL.md descriptions, CHANGELOG fragments, references/ files) at `epic-head-SHA`. This is the chicken-and-egg cross-check from [smoke-test-harness §5.1](/COPAAA/issues/COPAAA-7#document-smoke-test-harness); subsequent epic-close passes rely on the verified anchor without re-doing the manual cross-check.

For the rationale behind each rule, the per-check Auditor-challenge surfaces, the explicit out-of-scope list, and worked R1–R4 examples (RECONCILED, DRIFT, STALE), read: `skills/doc-reconciliation/references/checks.md`
