---
name: gate-review
description: >
  Execute a CopperForge Plan, Test, Code, or Epic Gate — read the artifact and
  the Auditor's findings, write the structured gate-review document
  (identity / inputs / dispositions / decision / rationale / sources), and
  hand off the issue with one decision token from the closed ladder
  PASS / PASS-WITH-CONDITIONS / REDIRECT / BLOCK. Use this skill when you are
  QAM running a gate on a CopperForge issue and must record the decision in
  the gate-review document the Auditor and Dev will reference downstream.
---

# Gate Review

Use this skill when you are the QAM running a Plan, Test, Code, or Epic Gate on a CopperForge issue and must produce both the structured `gate-review` (or `code-gate-review`) document and the closure comment that pushes the issue forward. Decisions ride on the closed ladder `PASS | PASS-WITH-CONDITIONS | REDIRECT | BLOCK`; conditions are enumerated as concrete follow-up issue identifiers with `blockedByIssueIds` wired to the dependent work, never as free-text "QAM should later …" promises.

## Preconditions

- The gate type is named explicitly: `Plan Gate`, `Test Gate`, `Code Gate`, or `Epic Gate`. The gate's anchor issue identifier is known.
- The artifact under review has been read in this heartbeat: BA's `research-summary` (Plan Gate); QA's structural-test results T1–T7 per [research-summary §3.6](/COPAAA/issues/COPAAA-2#document-research-summary) (Test Gate); the four evidence-package artifacts A–D per [code-gate-evidence-shape §2 / §3](/COPAAA/issues/COPAAA-22#document-code-gate-evidence-shape) (Code Gate); the smoke-summary roll-up plus tag-protection evidence per [smoke-test-harness §5.1](/COPAAA/issues/COPAAA-7#document-smoke-test-harness) (Epic Gate).
- The Auditor's findings produced via the [auditor-challenge skill](/COPAAA/issues/COPAAA-14) have been posted on the gate issue, with each finding numbered by severity tier (`MAJOR 1`, `MAJOR 2`, `MINOR 1`, `NIT 1`, …) and accompanied by the Auditor's Disposition section. Gates without an Auditor pass are malformed; route the issue back to the Auditor before opening the gate.
- The acceptance criteria for the gate are read from the relevant issue's `plan` document or from the upstream gate-review document, so each disposition is testable against a written rule rather than personal preference.
- Determinative sources for citations are accessible: methodology subsection deep-links, comment ids on the gate issue, and (for Code Gate) 40-char commit SHAs resolvable on the canonical remote.

## Workflow

1. **Identify the gate.** State the gate type (`Plan Gate | Test Gate | Code Gate | Epic Gate`), the anchor issue identifier, the predecessor gate decision (if any) with a deep-link to its review document, and one sentence of methodology context (anchor gate vs follow-on, vertical-slice vs parallelized).

2. **Pick the document key (closed list).** Default key is `gate-review`. If the gate is a Code Gate AND a `gate-review` document already exists on this issue (from any prior gate — Test Gate is the ratified case at [COPAAA-12](/COPAAA/issues/COPAAA-12), but Plan-Gate-then-Code-Gate or Epic-Gate-then-Code-Gate on the same issue diverts the same way), write to `code-gate-review` instead and add a `**Document-key note (process):**` line in §1 explaining the divergence (per the two-doc precedent). Plan and Epic Gates always use `gate-review`. Inventing keys outside this list (`test-gate-review`, `plan-gate-review`, `epic-gate-review`) is malformed.

3. **Build §1 Gate identity.** Bullet list with: gate type, issue under review (deep-link), scope (the decision space — `pass | pass-with-conditions | redirect | block`), one sentence of methodology context.

4. **Build §2 Inputs reviewed.** Markdown table, columns `Input | Link`. Required rows for every gate type: the artifact under review (with size on disk where applicable), the Auditor challenge pass (deep-link to comment id), the QAM pre-pass note (if QAM independently re-checked the evidence), and the upstream gate-review document if any. For Code Gate, also include `Base ref` and `Head ref` rows with literal 40-char SHAs (branch names, tag names, or `current` are malformed).

5. **(Code Gate only) Add §3 Push-pending disclosure when applicable.** If the head SHA is local-only or the push status to the canonical remote is anything other than "pushed", state it explicitly under a `## 3. Push-pending disclosure` heading so the Auditor can elect (a) require-push-before-signoff or (b) verify-from-literal-patches. Per the [COPAAA-12](/COPAAA/issues/COPAAA-12) anchor-gate hand-validation precedent, this disclosure is binding when relevant and silence is malformed.

6. **(Code Gate only) Place the §4 Evidence section.** Reproduce the four evidence-package artifacts A–D verbatim under the literal level-2 heading `## 4. Evidence (versioning-policy §4.5 / COPAAA-22)`, using the per-artifact level-3 sub-headings the [evidence-package skill](/COPAAA/issues/COPAAA-12) prescribes. Do NOT paraphrase the Dev's evidence — the Auditor's challenge surface is the literal artifact text.

7. **Build the Auditor findings + dispositions section.** Number the section appropriately (§3 for Plan/Test/Epic Gates; §9 for a Code Gate that also carries §3 push-pending and §4 evidence per [COPAAA-12](/COPAAA/issues/COPAAA-12) precedent). One subsection per numbered Auditor finding (`### C1 — [SEVERITY] <claim title>`). Every subsection MUST contain:

   ```markdown
   > Source: [Auditor <comment-id>, <Cn> §](/COPAAA/issues/<gate-issue-identifier>#comment-<comment-id>)

   **Finding.** <one or two sentences restating the claim and basis without re-quoting the auditor's full block — the source link carries the full text>

   **Disposition: <ACCEPTED | REJECTED | DEFERRED> — <one-sentence rule applied>.**

   <one or two paragraphs of disposition reasoning, including any concrete follow-up issue identifier filed (with `priority`, `assigneeAgentId`, and `blockedByIssueIds` already wired)>
   ```

   Severity-to-disposition rule (mechanical, not stylistic):
   - `BLOCKER` ACCEPTED → the gate cannot pass even with conditions; force decision to `BLOCK` and route the artifact back to its author for redo. REJECTED requires written justification linking to a determinative source that contradicts the Auditor's basis.
   - `MAJOR` ACCEPTED → becomes a binding condition on `PASS-WITH-CONDITIONS`; file a follow-up issue at workflow step 8. REJECTED requires written justification.
   - `MINOR` / `NIT` ACCEPTED → non-blocking follow-up filed (low priority) or folded into an existing follow-up; documented but does not pause downstream work. REJECTED is a one-line concur-or-leave note.

   QAM may invent neither severities the Auditor did not record nor dispositions outside the `ACCEPTED | REJECTED | DEFERRED` ladder. To upgrade or downgrade a severity, route a fresh challenge through the Auditor at the next iteration; mid-gate severity rewriting is malformed.

8. **File binding-condition follow-ups before writing §Decision.** For every ACCEPTED `MAJOR` and every ACCEPTED `BLOCKER` that converts to a redo, file a concrete follow-up issue: title that names the condition, `priority` (`high` for binding `MAJOR`, `critical` for `BLOCKER` redo), `assigneeAgentId` set at filing time, `parentId` set to the relevant epic, and — critically — set `blockedByIssueIds` on the dependent downstream issue(s) so Paperclip's automatic wake-on-blockers-resolved fires when the follow-up reaches `done`. Free-text "blocks COPAAA-X" comments without the dependency wired are malformed and silently break the wake.

9. **Build the Decision section.** Single, unhedged decision token in **bold** from the closed ladder `PASS | PASS-WITH-CONDITIONS | REDIRECT | BLOCK`. Hedged decisions (`PASS-leaning`, `BLOCK or REDIRECT`, `borderline PASS-WITH-CONDITIONS`) are gate-failing on their own. For `PASS-WITH-CONDITIONS`:
   - Enumerate every binding condition by issue identifier (`[COPAAA-NN](/COPAAA/issues/COPAAA-NN)`) with the `blockedByIssueIds` linkage already wired per workflow step 8.
   - Enumerate non-blocking observations under a separate `**Non-blocking notes:**` line so they do not conflate with conditions.
   - State which downstream issue is blocked by which follow-up so the Auditor can verify the wake will fire.

10. **Build the Rationale section.** Short numbered list of the load-bearing claims that justify the decision (typically 2–4). Each claim points back to a concrete artifact: an Auditor finding number, a structural-test row, an evidence-package artifact, or a methodology subsection. Vague rationale ("the test passed and we feel good about it") is itself a Code-Gate or Epic-Gate flag against this gate-review at the next iteration.

11. **Build the Sources section.** Bulleted list of every authoritative reference touched in the decision: methodology subsection deep-links, predecessor gate-review documents, file paths with 40-char SHAs (Code Gate), comment ids of the Auditor pass and the QAM pre-pass. The Sources list must allow a reader to reproduce the gate review without re-reading the issue thread.

12. **Post the gate-review document.** `PUT /api/issues/{issueId}/documents/<doc-key>` with the assembled body, `format: "markdown"`, and the title `<Gate Type> review — <issue-identifier> (<scope>)`. If the document already exists, fetch it first and pass the latest `baseRevisionId`.

13. **Post the closure comment.** Short markdown comment on the gate issue with: the decision token in **bold**, the document deep-link (`/<prefix>/issues/<issue-identifier>#document-<doc-key>`), the enumerated binding-condition issue identifiers (if any), and the routing line (`Reassign to <next-role>` for the next step in the per-skill internal sequence — e.g., on Test Gate `PASS`, route to Dev for evidence-package preparation; on Code Gate `PASS`, route to CTO/CEO for merge-and-tag).

14. **Reassign and update status.** PATCH the issue with the appropriate status: `in_progress` for `REDIRECT` (back to upstream author), `in_review` if returning to a different reviewer, `done` only when the gate decision closes the issue itself (a standalone gate-only issue with no follow-on work). For `PASS-WITH-CONDITIONS` where the issue continues, the issue stays with the dev/QA role responsible for the next internal step, NOT with QAM. Always re-send the full `blockedByIssueIds` array on the PATCH so a status-only update does not wipe blockers.

## Quality bar

- Exactly one decision token from the closed ladder `PASS | PASS-WITH-CONDITIONS | REDIRECT | BLOCK`. Hedges are gate-failing.
- The decision token in the closure comment matches the Decision section in the document. Drift between the comment and the document is itself a finding at the next gate.
- Every binding condition is a filed follow-up issue with `priority`, `assigneeAgentId`, and `blockedByIssueIds` wired at filing time. Free-text "QAM should later …" promises are malformed.
- Every numbered Auditor finding has a Disposition. Silence on a finding is malformed; QAM cannot omit a finding from §3 / §9 and rely on the reader to infer concurrence.
- Every Disposition cites the severity-to-disposition rule applied so the decision is testable against the Auditor's challenge surface, not against personal preference.
- Severity reuse only — QAM never invents severities (`BLOCKER | MAJOR | MINOR | NIT`) the Auditor did not record. Mid-gate severity rewriting is malformed; route a fresh Auditor pass instead.
- For Code Gate, every SHA cited is exactly 40 hex chars and identical across the document (Base ref / Head ref / Artifact B / Artifact C). SHA mismatches are gate-failing.
- For Code Gate, the Evidence section reproduces Artifacts A–D verbatim. Paraphrasing the Dev's evidence collapses the Auditor's challenge surface.
- For the FIRST per-skill Code Gate of the v1 epic ([COPAAA-12](/COPAAA/issues/COPAAA-12) — the methodology anchor) the gate-review MUST embed the Auditor hand-validation cross-check from [smoke-test-harness §5.1](/COPAAA/issues/COPAAA-7#document-smoke-test-harness). Subsequent skills' Code Gates rely on the verified anchor without re-doing the manual cross-check.
- Document key is one of `gate-review` or `code-gate-review` (per workflow step 2). Inventing other keys breaks the two-doc precedent ratified at [COPAAA-12](/COPAAA/issues/COPAAA-12) and the cross-issue heuristic that "the gate-review document lives at `#document-gate-review` unless a prior gate already wrote there" (Test Gate is the ratified case; Plan-Gate-then-Code-Gate or Epic-Gate-then-Code-Gate on the same issue diverts identically).
- The Auditor produces findings; QAM produces the decision. QAM never edits the Auditor's challenge text in place. Disagreement with a finding is recorded as a REJECTED Disposition with cited basis, never as a silent rewrite.
