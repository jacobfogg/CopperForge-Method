# Auditor finding disposition — template + severity rules

This reference loads only when QAM is dispositioning Auditor findings inside workflow step 6. It is a self-contained decision rubric and is not required to load on every gate-review invocation, so it lives outside `SKILL.md` per the [research-summary §3.4](/COPAAA/issues/COPAAA-2#document-research-summary) ~6 KB soft cap (criterion 1 — schema / decision matrix that does not need to load every invocation).

## Per-finding markdown block

One subsection per numbered Auditor finding (`### C1 — [SEVERITY] <claim title>`). Every subsection MUST contain:

```markdown
> Source: [Auditor <comment-id>, <Cn> §](/COPAAA/issues/<gate-issue-identifier>#comment-<comment-id>)

**Finding.** <one or two sentences restating the claim and basis without re-quoting the auditor's full block — the source link carries the full text>

**Disposition: <ACCEPTED | REJECTED | DEFERRED> — <one-sentence rule applied>.**

<one or two paragraphs of disposition reasoning, including any concrete follow-up issue identifier filed (with `priority`, `assigneeAgentId`, and `blockedByIssueIds` already wired)>
```

## Severity-to-disposition rule (mechanical, not stylistic)

- `BLOCKER` ACCEPTED → the gate cannot pass even with conditions; force decision to `BLOCK` and route the artifact back to its author for redo. REJECTED requires written justification linking to a determinative source that contradicts the Auditor's basis.
- `MAJOR` ACCEPTED → becomes a binding condition on `PASS-WITH-CONDITIONS`; file a follow-up issue at workflow step 7. REJECTED requires written justification.
- `MINOR` / `NIT` ACCEPTED → non-blocking follow-up filed (low priority) or folded into an existing follow-up; documented but does not pause downstream work. REJECTED is a one-line concur-or-leave note.

QAM may invent neither severities the Auditor did not record nor dispositions outside the `ACCEPTED | REJECTED | DEFERRED` ladder. To upgrade or downgrade a severity, route a fresh challenge through the Auditor at the next iteration; mid-gate severity rewriting is malformed.

## Sources

- [auditor-challenge skill](/COPAAA/issues/COPAAA-14) — the challenge surface that produces the numbered findings dispositioned here.
- [research-summary §3.4](/COPAAA/issues/COPAAA-2#document-research-summary) — the ~6 KB soft cap and load-on-demand criterion this file satisfies.
