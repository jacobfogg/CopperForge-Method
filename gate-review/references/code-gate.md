# Code Gate — additional sections (loaded only on Code Gate)

This reference loads on Code Gate only. Plan, Test, and Epic Gate runs do not need it. Splitting it out keeps `SKILL.md` under the [research-summary §3.4](/COPAAA/issues/COPAAA-2#document-research-summary) ~6 KB soft cap (criterion 1 — long worked example that does not need to load every invocation).

The two sections below are inserted between workflow step 4 (`§2 Inputs reviewed`) and the Auditor findings + dispositions section. Number them §3 and §4 on a Code Gate; on a Test Gate that is later replaced by a Code Gate, follow the [COPAAA-12](/COPAAA/issues/COPAAA-12) two-doc precedent and write the Code Gate evidence to the `code-gate-review` document key (per workflow step 2 in `SKILL.md`).

## §3 Push-pending disclosure (Code Gate only)

If the head SHA is local-only or the push status to the canonical remote is anything other than "pushed", state it explicitly under a `## 3. Push-pending disclosure` heading so the Auditor can elect (a) require-push-before-signoff or (b) verify-from-literal-patches. Per the [COPAAA-12](/COPAAA/issues/COPAAA-12) anchor-gate hand-validation precedent, this disclosure is binding when relevant and silence is malformed.

## §4 Evidence section (Code Gate only)

Reproduce the four evidence-package artifacts A–D verbatim under the literal level-2 heading `## 4. Evidence (versioning-policy §4.5 / COPAAA-22)`, using the per-artifact level-3 sub-headings the [evidence-package skill](/COPAAA/issues/COPAAA-12) prescribes. Do NOT paraphrase the Dev's evidence — the Auditor's challenge surface is the literal artifact text.

## Code-Gate-only quality bar additions

These bars apply only on Code Gate; the gate-agnostic quality bar in `SKILL.md` is the rest.

- For Code Gate, every SHA cited is exactly 40 hex chars and identical across the document (Base ref / Head ref / Artifact B / Artifact C). SHA mismatches are gate-failing.
- For Code Gate, the Evidence section reproduces Artifacts A–D verbatim. Paraphrasing the Dev's evidence collapses the Auditor's challenge surface.
- For the FIRST per-skill Code Gate of the v1 epic ([COPAAA-12](/COPAAA/issues/COPAAA-12) — the methodology anchor) the gate-review MUST embed the Auditor hand-validation cross-check from [smoke-test-harness §5.1](/COPAAA/issues/COPAAA-7#document-smoke-test-harness). Subsequent skills' Code Gates rely on the verified anchor without re-doing the manual cross-check.

## Sources

- [code-gate-evidence-shape §2 / §3](/COPAAA/issues/COPAAA-22#document-code-gate-evidence-shape) — the four-artifact A–D shape referenced above.
- [COPAAA-12](/COPAAA/issues/COPAAA-12) — anchor-gate hand-validation precedent and two-doc (gate-review / code-gate-review) ratification.
- [evidence-package skill](/COPAAA/issues/COPAAA-12) — per-artifact level-3 sub-heading rubric.
