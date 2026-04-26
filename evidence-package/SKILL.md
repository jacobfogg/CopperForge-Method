---
name: evidence-package
description: >
  Assemble the per-skill Code-Gate evidence package — semver-bump enum,
  frontmatter description before/after diff, references/ rename-or-removal
  grep, and CHANGELOG fragment — placed under the canonical heading in the
  gate's review document so the Auditor can challenge any claim against a
  literal artifact instead of re-reading the diff. Use this skill when
  preparing the evidence section of a CopperForge per-skill Code Gate, or
  when assembling the v1.0.0 epic-close Code-Gate evidence package.
---

# Evidence Package

Use this skill when you are about to open a CopperForge per-skill Code Gate (or the v1.0.0 epic-close gate) and must record the evidence the Auditor will challenge. It produces four mechanical artifacts (A–D) per skill, places them under a fixed heading in the gate's `gate-review` document, and adds the smoke-summary and tag-protection artifacts at the epic-close gate. The methodology rationale, source policy, Auditor-challenge surfaces, and worked examples are in `references/per-skill.md`; this body specifies the workflow.

## Preconditions

- The change is staged for a Code Gate; both the `base` commit the gate reviews against and the proposed `head` commit resolve to 40-char SHAs.
- The gate issue exists and you can create or update its `gate-review` document.
- You have read [versioning-policy §4.1, §4.2, §4.5](/COPAAA/issues/COPAAA-6#document-versioning-policy) and the artifact contract [COPAAA-22 → code-gate-evidence-shape](/COPAAA/issues/COPAAA-22#document-code-gate-evidence-shape). The skill enforces that contract; this document does not redefine it.
- Epic-close gate only: smoke-test harness has run and `summary.json` is on disk; tag-protection (or its enumerated upstream blocker per [COPAAA-23](/COPAAA/issues/COPAAA-23)) is captured.

## Workflow

1. **Resolve scope.** Identify every CopperForge skill changed between `base` and `head`. Each skill in scope gets its own complete A–D set; artifacts MUST NOT be shared across skills even when "the change was the same."
2. **Resolve refs.** Capture both 40-char commit SHAs (`base-ref-SHA`, `head-ref-SHA`). Branch names, tag names, or "current" are malformed.
3. **Artifact A — Semver bump enum classification.** Pick exactly one of `MAJOR | MINOR | PATCH` per skill. Hedged tokens ("MINOR-leaning", "MAJOR or MINOR") are gate-failing on their own. Quote one row's trigger text from versioning-policy §4.1 verbatim. Apply §4.2's five operational-breaking bullets in one sentence. For `MAJOR`, enumerate every changed/removed agent invocation or instruction; for `MINOR`/`PATCH`, write the literal token `N/A`. Use the literal block:

   ```
   SEMVER_BUMP: MAJOR | MINOR | PATCH
   §4.1_ROW: "<exact trigger text from versioning-policy §4.1>"
   §4.2_BREAKING_CHECK: <one sentence applying §4.2's five bullets>
   MAJOR_INSTRUCTION_LIST: <enumerated list, or N/A>
   ```

4. **Artifact B — Frontmatter description before/after diff.** Always present, even when description is unchanged. Use a code-fenced diff with both 40-char SHAs in the file headers. For an unchanged case, leave the body empty AND add the literal sentence `description unchanged at <head SHA>` directly below the fence. For a non-empty diff, classify the direction with exactly one of `WIDENED`, `NARROWED`, `REWRITTEN_SAME_TRIGGER_SET` directly below the fence. For `REWRITTEN_SAME_TRIGGER_SET`, also write `triggers_before: [...] / triggers_after: [...]` showing identical sets.
5. **Artifact C — `references/` rename-or-removal grep.** Run `git diff --name-status <base-SHA> <head-SHA> -- '<skill-dir>/references/'` and capture the verbatim output even if empty (empty IS the evidence). If there are `R` (rename) or `D` (delete) lines, then for every old path, run `git grep -n -F "$f" <head-SHA> -- '<skill-dir>/SKILL.md'` and reproduce the verbatim output. Both commands' SHAs MUST match Artifact B's SHAs. A non-empty `git grep` result (SKILL.md still cites a removed/renamed path) is gate-failing as a broken-reference bug, separate from the bump.
6. **Artifact D — CHANGELOG.md entry fragment.** Paste-ready markdown — no `<TBD>`, `XXX`, or unresolved placeholders other than the four named below, all of which MUST be filled in:

   ```markdown
   ### `<skill-key>` <SEMVER_BUMP> — <one-line customer-observable summary>

   - <bullet describing the change in customer-observable terms>
   - Code Gate: [<gate-issue-identifier>](/COPAAA/issues/<gate-issue-identifier>)
   - Bump rationale: §4.1 row "<exact trigger text from §4.1>"
   ```

   `<skill-key>` is the canonical slug (e.g., `evidence-package`), not the human title. `<SEMVER_BUMP>` MUST match Artifact A's token literally. `<gate-issue-identifier>` MUST be the per-skill Code Gate issue's identifier (linking to the parent epic instead is malformed). The §4.1 row text MUST match Artifact A's `§4.1_ROW` verbatim.

7. **Place artifacts in the gate-review doc.** Inside the gate's `gate-review` document, create exactly this level-2 heading:

   ```
   ## Evidence (versioning-policy §4.5 / COPAAA-22)
   ```

   Under it, add Artifacts A→D in order, each with a level-3 sub-heading (`### Artifact A — Semver bump enum classification`, `### Artifact B — …`, etc.). If the gate covers multiple skills, give each skill its own complete A–D set under a level-3 heading naming the skill key.

8. **Cross-consistency self-check.** Before handing the gate off, verify:
   - Direction-token ↔ bump pairing: `NARROWED` ↔ `MAJOR`; `WIDENED` ↔ `MINOR`; `REWRITTEN_SAME_TRIGGER_SET` ↔ `PATCH`. Empty diff is compatible with any bump (then the rationale comes from a different §4.1 row).
   - SHAs match across Artifacts B and C.
   - Artifact D's `<skill-key>`, `<SEMVER_BUMP>`, gate-issue identifier, and §4.1 row text match Artifacts A–C.
   Any mismatch is gate-failing; fix before submitting.

9. **Epic-close gate only.** When the gate is the v1.0.0 publish gate, also produce the smoke-summary roll-up and the tag-protection artifact described in `references/epic-close.md`. Per-skill Code Gates do NOT need these.

## Quality bar

- One bump per skill, no hedged tokens. "Approximately MINOR" is gate-failing.
- Every artifact is literal (output captured, not paraphrased). "I checked and it's fine" without command output is malformed.
- Every SHA cited in Artifacts B and C is exactly 40 hex chars and identical across artifacts.
- Artifact A's `§4.1_ROW` text appears verbatim in Artifact D's bump-rationale bullet.
- Artifact D is paste-ready: no remaining `<…>`, `XXX`, or `TBD` placeholders.
- Every claim that survives this check must be Auditor-falsifiable in one sentence: "Artifact X claims Y, but Artifact Z says W." If you can't articulate what would falsify each claim, the artifact is too vague — rewrite it.
- For the FIRST per-skill Code Gate of the epic ([COPAAA-12](/COPAAA/issues/COPAAA-12), this skill itself): the package MUST be hand-validated by the Auditor against the underlying inputs (test results, diff, doc-update reference). This is the chicken-and-egg cross-check from [smoke-test-harness §5.1](/COPAAA/issues/COPAAA-7#document-smoke-test-harness); subsequent skills' Code Gates rely on the verified anchor without re-doing the manual cross-check.

For the rationale behind each rule, the per-artifact Auditor-challenge surfaces, the out-of-scope list, and worked per-skill and multi-skill examples, read: `skills/evidence-package/references/per-skill.md`

For the additional artifacts the v1.0.0 epic-close Code Gate requires (smoke-summary roll-up + tag-protection per [COPAAA-23](/COPAAA/issues/COPAAA-23)), read: `skills/evidence-package/references/epic-close.md`
