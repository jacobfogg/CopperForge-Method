---
name: test-change-request
description: >
  File a structured request from Dev to QAM proposing a change to a specific
  test or test rubric — naming the one test under challenge, classifying the
  defect basis with one enum token, quoting the current rule text verbatim,
  and proposing a concrete replacement — while distinguishing test-rubric
  concerns from acceptance-criteria concerns so the request routes to the
  right reviewer. Use this skill when a Dev believes a structural test or
  rubric rule is wrong, ambiguous, or mis-scoped and wants QAM to dispose
  of the change before (or alongside) the current gate decision.
---

# Test Change Request

Use this skill when you, as Dev, believe a structural test or test-rubric rule is wrong, ambiguous, or mis-scoped and you want QAM to dispose of the change through a reviewed request. It encodes the request shape (one test, one defect-basis token, one proposed change), the test-vs-AC distinction that determines routing, and the rule that the current gate proceeds against the rubric as it stood when QAM convened — accepted changes apply prospectively, never retroactively.

## What this skill is, and what it is not

This skill is **for**:

- A Dev who hit a test rule they believe is structurally wrong (false-positive, false-negative, ambiguous, or mis-scoped) and wants QAM to consider changing the rule.
- A request that targets the **rubric** — the rule text or test enumerated in a research-summary or QA runner — not the result of running the rubric on a specific commit.

This skill is **not for**:

- Filing a defect against a specific commit ("the test failed on my SHA"). The gate review handles that; if the test ruled correctly, fix the code.
- Changing acceptance criteria. AC concerns route to the BA / CTO via the issue's description, not to QAM via this skill.
- Re-arguing a gate decision after it landed. A REDIRECT or PASS-WITH-CONDITIONS is a final disposition; a follow-up ticket is the right vehicle, not a rubric-change request.
- Working around a rule silently in a future commit. Silent workarounds are gate-failing; the request is the audit trail that protects you.

## Preconditions

- A specific test or test-rubric rule exists in writing (a research-summary section, a QA runner check, a SKILL.md quality-bar bullet) that you can quote verbatim.
- You can name the canonical identifier of the test or rule (e.g., `T4` from [research-summary §3.6](/COPAAA/issues/COPAAA-2#document-research-summary), or the rubric-doc heading and bullet).
- You have asked yourself: "Is this a test-rubric concern or an acceptance-criteria concern?" — and the answer is test-rubric (see §"Distinguishing test from AC concerns" below).
- A gate issue exists where the comment will land (typically the per-skill build issue currently in Test Gate or Code Gate).

## Workflow

1. **Identify the one test under challenge.** Name it by canonical identifier — e.g., `T4` (frontmatter shape) from [research-summary §3.6](/COPAAA/issues/COPAAA-2#document-research-summary), or `quality-bar bullet 3` from a specific `SKILL.md`. One request targets exactly one test or rule. Bundling multiple tests into one request is malformed — file one request per rule so QAM can dispose of each independently.

2. **Quote the current rule text verbatim.** Paste the rule's text from its canonical source as a fenced block, with a citation that includes the document key and the section / bullet identifier. Paraphrasing the rule is gate-failing because QAM cannot dispose of a request whose target is fuzzy. If the rule is split across two sources (e.g., a research-summary §3.6 row plus a runner-check function), quote both.

3. **Classify the defect basis.** Pick exactly one token:

   - `FALSE_POSITIVE` — rule fires on a SKILL.md that is structurally correct.
   - `FALSE_NEGATIVE` — rule passes a SKILL.md that is structurally wrong.
   - `AMBIGUOUS_RULE` — rule text admits two reasonable interpretations and Dev/QA cannot resolve which by reading the rubric alone.
   - `MISSING_RULE` — rubric does not check a property it should (no current rule fires; one should be added).
   - `OVER_RESTRICTIVE_SCOPE` — rule blocks a SKILL.md that meets the methodology's stated intent.
   - `UNDER_RESTRICTIVE_SCOPE` — rule lets a SKILL.md through that violates the methodology's stated intent.

   Hedged tokens ("FALSE_POSITIVE-leaning", "AMBIGUOUS_RULE or MISSING_RULE") are gate-failing — pick one and defend it. If two tokens both seem to fit, file two separate requests.

4. **Provide the supporting evidence.** For `FALSE_POSITIVE` / `FALSE_NEGATIVE`, paste the exact SKILL.md fragment that triggered (or failed to trigger) the rule, plus the runner output if any. For `AMBIGUOUS_RULE`, write the two interpretations and what each would imply for the SKILL.md under review. For `MISSING_RULE`, name the structural property that is currently unchecked. For scope tokens, quote the methodology-intent sentence the rule contradicts. Evidence that QAM cannot reproduce by reading your comment alone is malformed.

5. **Propose the replacement as a concrete diff.** Write the rule text as it currently reads, then the rule text as you propose it should read, in a fenced before/after block. "Make T4 stricter" is gate-failing; "replace `description: present` with `description: present and contains an allow-list substring`" is reviewable. For `MISSING_RULE`, write only the proposed new rule (no `before` block). The proposed text MUST be runnable by the same rubric mechanics — do not propose a rule that requires a runner the QA team has not built.

6. **Confirm the test-vs-AC distinction.** State explicitly that this is a test-rubric concern, not an AC concern, and write one sentence of why. The check: an AC concern would change the SKILL.md's stated purpose or quality bar; a test-rubric concern would change how the existing purpose is verified. If you cannot make this statement honestly, the request is misrouted — file an AC-change request against the parent issue's description instead.

7. **State the requested disposition window.** Pick one:

   - `BEFORE_CURRENT_GATE` — the current gate cannot fairly proceed without resolving this; QAM must dispose before convening.
   - `ALONGSIDE_CURRENT_GATE` — gate may proceed under the current rubric; QAM disposes in parallel and the change applies prospectively.
   - `POST_GATE` — current gate proceeds and closes under the current rubric; the request enters QA's backlog as a methodology-improvement candidate.

   Default to `ALONGSIDE_CURRENT_GATE` unless the rule under challenge is the basis for the gate's likely failure verdict — then `BEFORE_CURRENT_GATE`. Per the [COPAAA-31](/COPAAA/issues/COPAAA-31) onboarding rule, accepted changes are **always** prospective — QAM cannot retroactively re-rule a gate that has already landed.

8. **File on the gate issue and route to QAM.** Post the request as a comment on the per-skill build issue currently at gate (or, for cross-skill rule changes, on the parent epic). Reassign to the QAM if the comment changes the gate's critical path; otherwise leave the assignee as-is and ping QAM by name. The comment is the audit trail; QAM's reply (accept / reject / refine) is the disposition.

## Distinguishing test from AC concerns

Test-rubric concerns and acceptance-criteria concerns share surface area but route differently. Use this sieve:

- **Does the change alter what "done" means for the SKILL.md author?** If yes, it is an AC concern — route to BA / CTO via the issue description. If no, it is a test concern.
- **Does the change alter how a fixed definition of "done" is checked?** If yes, it is a test concern — route to QAM via this skill. If no, you do not yet have a request.
- **Would two reviewers reading the same SKILL.md disagree on whether it passes?** That is an `AMBIGUOUS_RULE` test concern, not an AC concern — file via this skill, not via a description revision.

If both apply (the rule under challenge is also load-bearing on the AC), file **two** requests — one to BA / CTO for the AC change, one to QAM for the rubric change — and link them. Smuggling an AC change inside a rubric-change request is gate-failing because the wrong reviewer disposes of the wrong question.

## Quality bar

- **One test per request, one defect-basis token, one proposed change.** Bundles are malformed.
- **The rule text is quoted verbatim with a citation that resolves to a document key and section / bullet.** Paraphrasing is gate-failing.
- **The defect-basis token is one of the six enum values, no hedging.** "FALSE_POSITIVE-leaning" is malformed.
- **The proposed change is concrete enough that QAM could land it as a diff.** "Make it stricter" is gate-failing.
- **The test-vs-AC distinction is stated explicitly with one sentence of why.** Implicit routing is malformed.
- **The disposition window is one of `BEFORE_CURRENT_GATE` / `ALONGSIDE_CURRENT_GATE` / `POST_GATE`.** Unmarked requests default to `ALONGSIDE_CURRENT_GATE` and may not block the current gate.
- **Accepted changes apply prospectively only.** Per [COPAAA-31](/COPAAA/issues/COPAAA-31), an in-flight gate runs against the rubric as it stood when QAM convened; the accepted change lands in a follow-up ticket against the rubric's canonical document.
- **The request is Auditor-falsifiable in one sentence:** "Request claims rule X behaves as Y, but the cited source says Z." If you cannot articulate what would falsify your claim, the request is too vague — rewrite before filing.

## Out of scope

- Defects against a specific commit. The gate review and the underlying code are the right vehicles.
- Acceptance-criteria changes. Route to BA / CTO via the issue description, not to QAM via this skill.
- Disputes about a closed gate's verdict. File a follow-up ticket; do not reopen via a rubric-change request.
- Building or modifying the runner that executes the rubric. The runner is QA-owned; this skill files requests against the rules it executes, not against the runner's code.
