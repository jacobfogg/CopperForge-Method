---
name: auditor-challenge
description: >
  Produce the Auditor's structured findings on a CopperForge Plan, Test, or
  Code Gate — one block per claim challenged, each carrying the four
  required fields (claim, basis, evidence-or-alternative, severity) so QAM
  can route every finding to a discharge action without re-reading the
  source. Use this skill when you are the Auditor reviewing a gate artifact
  and must record findings in a form QAM can act on directly.
---

# Auditor Challenge

Use this skill when you are the Auditor on a CopperForge Plan, Test, or Code Gate and must record findings on a specific gate artifact (a research-summary subsection, a structural-test result row, or an evidence-package artifact A–D). Each finding produced by this skill is a four-field block — Claim challenged, Basis, Evidence requested OR alternative proposed, Severity — placed in the gate's review thread or `gate-review` document so QAM can map every finding to a discharge action without re-reading the source.

## Preconditions

- The gate type is named explicitly: `Plan Gate`, `Test Gate`, or `Code Gate`. The gate's anchor issue identifier is known.
- The source artifact under review has been fetched and read in this heartbeat. For Plan Gates that is the BA's `research-summary` document; for Test Gates the QA structural-test results (T1–T7 per [research-summary §3.6](/COPAAA/issues/COPAAA-2#document-research-summary)); for Code Gates the four evidence-package artifacts A–D produced via the [evidence-package skill](/COPAAA/issues/COPAAA-12).
- The acceptance criteria for the gate are read from the relevant issue's `plan` or `gate-review` document so each claim can be tested against a written rule, not against personal preference.
- Determinative sources for citations are accessible: file paths with line numbers (importer source, smoke-test harness), 40-char commit SHAs, and document deep-links of the form `/COPAAA/issues/<id>#document-<key>`.

## Workflow

1. **State concurrence first.** Open the audit with an explicit concurrence summary that names which top-level claims you accept without challenge. A bare list of challenges with no concurrence line silently implies the Auditor rejects everything not mentioned, which is malformed; QAM cannot infer concurrence from omission.

2. **Enumerate scope.** List the claims subject to audit (Plan Gate: `research-summary` subsection numbers; Test Gate: T1–T7 result rows; Code Gate: Artifacts A–D). For each scoped claim, decide concur or challenge. Record both decisions explicitly so the audit covers the full surface, not a sampled subset.

3. **Produce one challenge block per challenged claim.** Each block MUST contain all four fields in this exact order, and MUST use the literal labels below:

   ```markdown
   **Claim challenged:** "<verbatim quote from the source artifact, OR a paraphrase prefixed `paraphrase of` followed by the source location>"
   **Basis:** <citation to a determinative source that demonstrates the claim is wrong, incomplete, or unverifiable>
   **Evidence requested OR alternative proposed:** <exactly one — either a specific verifiable artifact you require to discharge the challenge, or a proposed replacement wording/action the author can adopt>
   **Severity:** <BLOCKER | MAJOR | MINOR | NIT>
   ```

4. **Pick severity from the closed ladder.**
   - `BLOCKER` — the gate cannot pass even with conditions; the artifact must be redone before the gate can issue a decision. Use only when a load-bearing claim is wrong on its face.
   - `MAJOR` — the gate may pass with conditions; closure requires the listed evidence or alternative to be discharged before downstream work begins. Numbered MAJORs (`MAJOR 1`, `MAJOR 2`, …) become the conditions on QAM's `pass with conditions` decision.
   - `MINOR` — the gate passes; the challenge is filed as a non-blocking follow-up (a comment on the source artifact's next revision pass, or a separate issue with `priority: low`). MINORs do not pause downstream work.
   - `NIT` — advisory only; the author may take or leave the suggestion. Use sparingly; never use NIT for anything that could affect downstream behavior or another agent's heartbeat decisions.

   Hedged severities (`MINOR-leaning`, `MAJOR or MINOR`, `borderline BLOCKER`) are not allowed. Pick one. If the claim genuinely spans two severities, split it into two challenge blocks, one per severity.

5. **Number challenges by severity tier.** Within a single audit, number challenges as `MAJOR 1`, `MAJOR 2`, …; `MINOR 1`, `MINOR 2`, …; `NIT 1`, …. Numbers are stable for the lifetime of the gate so QAM, BA, Dev, and downstream gates can reference each finding by `<gate-issue-identifier> MAJOR 1` without re-quoting the body.

6. **Field constraints — every block.**
   - **Claim challenged** is verbatim quoted from the source artifact (with quotation marks), or an explicit paraphrase prefixed `paraphrase of` plus the source location. Bare assertions about the artifact ("the section is unclear") are malformed.
   - **Basis** cites a determinative source: file path with line numbers, document deep-link with section reference, 40-char commit SHA, or literal test-runner output. "I think" / "this seems wrong" / unsourced opinion is malformed. A Basis tagged Assumption (because the cited source could not be verified in this heartbeat) MUST say so explicitly and the Severity MUST reflect the weaker support — typically capping at MINOR.
   - **Evidence requested OR alternative proposed** is exactly one of the two, never both. Asking for evidence and proposing an alternative in the same block is malformed; split into two challenges, one per ask.
   - **Severity** is one closed-ladder token with no prose modifiers and no compound severities.

7. **Close with a Disposition section.** Append a `### Disposition` block listing each numbered challenge with the recommended outcome — `concur close`, `pass with conditions` (one per MAJOR), `redirect`, or `block`. The Disposition is the input QAM uses to write the gate decision; QAM may adopt, narrow, or escalate it but never invent severities the Auditor did not record.

8. **Reassign to QAM.** The Auditor never sets the gate decision — QAM does. After posting the structured comment (or the corresponding section of the gate's `gate-review` document), reassign the gate issue to [QAM](/COPAAA/agents/qam) with a comment that links the audit and lists the numbered challenges so QAM's heartbeat lands on the structured findings, not free-text.

## Quality bar

- Every challenge block has all four fields with the literal labels in the order shown in step 3. A block missing any field is malformed and MUST be rewritten before posting.
- Every Basis cites a determinative artifact (file:line, doc deep-link, SHA, or test output). Unsupported opinions are not challenges.
- Every Severity is exactly one closed-ladder token (`BLOCKER`, `MAJOR`, `MINOR`, `NIT`). No hedges, no modifiers, no compound severities.
- Every block carries Evidence-requested OR alternative-proposed, never both. The XOR is mechanical, not stylistic — QAM's discharge routing depends on it.
- Concurrence on the rest of the artifact is stated explicitly in step 1. Silence is not concurrence; QAM treats unstated coverage as a gap, not as agreement.
- Numbering by severity tier (`MAJOR 1`, `MINOR 1`, `NIT 1`) is stable for the lifetime of the gate. Renumbering after the fact breaks downstream references in QAM dispositions and Dev rework comments.
- Tagging discipline (Verified vs Assumption) carries through Basis. An Assumption-only Basis is a weaker challenge; cap Severity at MINOR unless the Assumption is the artifact's own unsupported claim, in which case the asymmetry — Auditor's Assumption challenging the author's Verified — is itself the finding and should be filed as MAJOR with a Basis of `the cited source does not check out at the line range given`.
- The Auditor never writes the gate decision. QAM reads the Disposition section and decides; the Auditor's role is to surface findings in a form that makes QAM's decision routable in one pass.
