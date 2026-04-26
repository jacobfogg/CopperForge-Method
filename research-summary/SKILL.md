---
name: research-summary
description: >
  Produce a Plan-Gate-ready research summary that restates the deciding
  question, enumerates options with tradeoffs, recommends a path forward,
  surfaces risks, and tags every factual claim as Verified (with a primary
  citation) or Assumption (with the reason it cannot yet be verified). Use
  this skill when a Plan Gate (or any decision-grade review) requires a
  written research summary before scope decomposition or implementation
  can begin.
---

# Research Summary

Use this skill when you are about to convene a Plan Gate, or any decision-grade review that needs cited research, and you must produce the written summary the gate participants will challenge. It encodes a five-section template (Problem, Options, Recommendation, Risks, Sources), the Verified/Assumption tagging key that makes every claim Auditor-falsifiable, and the document-write + handoff mechanics. The methodology rationale and a worked example are linked at the end.

## Preconditions

- The deciding question is concrete enough to restate in one paragraph (not "should we modernize auth" but "should the v1 auth rewrite use library X, library Y, or stay on the current implementation").
- You can name the gate or decision forum that will consume the summary, and you understand its evidence rubric.
- You have access to authoritative sources: source files, API responses, vendor docs, prior tickets, or memory entries — not just secondhand summaries.
- A document slot exists on the gate's parent issue (typically the `research-summary` document key) where the finished summary will land.

## Workflow

1. **Restate the problem (§1).** Write a one-paragraph problem statement that names the deciding actor, the deciding question, the constraints, and the deadline. Tag the underlying facts as **(Verified — \<source\>)** or **(Assumption — \<reason\>)**. A summary that cannot restate the problem in one paragraph is not Plan-Gate-ready — narrow the question first.

2. **Enumerate options (§2).** List every option a reasonable reviewer would expect to see considered, including the status-quo option ("do nothing"). For each: a one-line summary, then the tradeoff bullets (cost, risk, time-to-ship, ergonomic implications). Reject options explicitly with a one-sentence rejection reason — silent omission is gate-failing because the Auditor cannot challenge what you did not write down.

3. **Recommend a path (§3).** Pick exactly one option as the recommendation. Spell out the operational details a downstream executor would need (directory layout, frontmatter shape, registration mechanics, etc.). Subdivide §3 into numbered subsections (§3.1, §3.2, …) when the recommendation has more than one mechanical surface area; this gives reviewers stable handles for revision packages. Every non-trivial claim in §3 carries a Verified or Assumption tag — untagged sentences are scaffolding only and assert nothing about the world.

4. **Surface risks and mitigations (§4).** Build a four-column table (`Risk | Likelihood | Impact | Mitigation`). Include risks the recommendation creates, not just the risks it avoids. A risk-table that contains only low/low rows is a smell — re-examine the recommendation honestly before publishing.

5. **Cite sources (§5).** List every source consulted, grouped by kind: filesystem (path + line range or commit SHA), API (endpoint + the timestamp the call returned), external doc (URL + revision/date), memory (cross-checked, never load-bearing). If a fact in §1–§4 rests only on a memory entry, re-tag it as **(Assumption)** until you have verified it against a primary source.

6. **Apply the Verified/Assumption tagging key.** Every factual sentence in §1–§4 carries one of two parenthetical tags at the end: **(Verified — \<one-line citation\>)** or **(Assumption — \<reason it cannot yet be verified\>)**. Declare the tagging key once near the top of the document. The Auditor's structural challenge is "for every untagged sentence, does the surrounding context prove it is scaffolding rather than a fact?" — write defensively.

7. **Write the document.** Use `PUT /api/issues/<gate-issue-id>/documents/<key>` with `format: markdown`. The conventional key is `research-summary`. Open with the H1 title, then a status line (issue link, parent epic link, author, status, last revised), then the tagging-key callout, then §1–§5 in order. If the document already exists, fetch it first and send the latest `baseRevisionId` on the PUT.

8. **Hand off to the gate.** Comment on the gate issue, link the document with `/<prefix>/issues/<id>#document-<key>`, and reassign to the gate convener (typically the QAM or BA depending on the gate). The summary is now an artifact the gate will challenge — every revision must update the `last revised` line and bump revision numbering in the status line so reviewers can diff revisions.

## Quality bar

- One Plan-Gate-ready research summary per deciding question — not one per option, not one per stakeholder.
- Every factual sentence in §1–§4 carries a (Verified — …) or (Assumption — …) tag. Untagged factual claims are gate-failing.
- Every option in §2 is named explicitly, including the status-quo option, and rejected options have a one-sentence rejection reason.
- Every source in §5 resolves to a primary artifact (file path + line range, commit SHA, API endpoint + timestamp, vendor URL). Memory references are cross-checked but never load-bearing.
- §3 is decision-ready: a reviewer who reads only §3 should know what to do next without needing to re-read §1–§2.
- Revisions update the document in place via `PUT .../documents/research-summary` with the latest `baseRevisionId`; do not append revision packages as separate documents.
- Every claim must be Auditor-falsifiable in one sentence: "§3.2 claims X, but the source at §5 says Y." If you cannot articulate what would falsify a claim, the claim is too vague — rewrite it.

For a worked Plan-Gate research summary that exercises every section of this template (Problem, Options A/B/C, Recommendation across §3.1–§3.6, Risks, and Sources grouped by kind), read the [COPAAA-2 `research-summary` document](/COPAAA/issues/COPAAA-2#document-research-summary) referenced from the parent epic.
