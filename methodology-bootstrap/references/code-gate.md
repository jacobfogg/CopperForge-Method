# Code Gate — methodology-bootstrap (install slice, B1 anchor)

This reference loads only at Code Gate for the `methodology-bootstrap` skill.
Plan, Test, and Epic Gate runs do not need it. Its presence satisfies the C5
criterion ([COPAAA-58 plan §4](/COPAAA/issues/COPAAA-58#document-plan)) — Code-Gate-only material separated from
`SKILL.md` per the [COPAAA-49](/COPAAA/issues/COPAAA-49) split convention.

## C1–C5 evidence criteria (Plan §4 / COPAAA-58)

At Code Gate for each `methodology-bootstrap` slice, the `code-gate-review`
document MUST contain the following five items. Slices B2
([COPAAA-69](/COPAAA/issues/COPAAA-69)) and B3 ([COPAAA-79](/COPAAA/issues/COPAAA-79)) accumulate additional columns onto this
reference file when their Code Gates run.

| # | Criterion | Notes |
|---|---|---|
| C1 | Head SHA on the canonical `jacobfogg/CopperForge-Method` remote. Anchor-gate bar: SHA MUST resolve on the remote ([COPAAA-19](/COPAAA/issues/COPAAA-19) B4). | Same repo as the skill library for this skill. |
| C2 | `methodology-bootstrap/SKILL.md` size (bytes) + sha256. | Per [COPAAA-83](/COPAAA/issues/COPAAA-83) Rule C/D frontmatter-key enumeration discipline. |
| C3 | Auditor disposition: PASS / PASS-WITH-CONDITIONS / FAIL with itemized M-issues. | Auditor challenges after Dev2 publishes the evidence package. |
| C4 | README row update for the skill in the canonical `README.md` skills index. B1 row records `install` only; B2 adds `upgrade` and B3 adds `drift-check` per Plan §4 C4 single-row convention. | [COPAAA-19](/COPAAA/issues/COPAAA-19) B4 rule (per-skill README row update is mandatory). |
| C5 | This file. | |

## Code-Gate-only quality bar additions

- **README row update** is mandatory at Code Gate per the [COPAAA-19](/COPAAA/issues/COPAAA-19) B4 rule
  (ratified 2026-04-26). B1's row records `install` as the only supported
  operation; B2 ([COPAAA-69](/COPAAA/issues/COPAAA-69)) and B3 ([COPAAA-79](/COPAAA/issues/COPAAA-79)) accumulate their operations
  onto the same row per Plan §4 C4. A missing README row is gate-failing.
- **Head SHA fidelity**: the head SHA cited in C1 MUST be verified to resolve
  on the canonical `jacobfogg/CopperForge-Method` remote before sign-off.
  A local-only SHA is gate-failing ([COPAAA-19](/COPAAA/issues/COPAAA-19) anchor-gate bar).
- **COPAAA-83 Rules C + D**: enumerate ALL frontmatter-key changes since the
  base (Rule C) and include the verbatim `git diff` in Artifact B (Rule D).

## Sources

- [COPAAA-58 plan §4](/COPAAA/issues/COPAAA-58#document-plan) — C1–C5 Code Gate criteria.
- [COPAAA-49](/COPAAA/issues/COPAAA-49) — split convention (Code-Gate-only material → `references/code-gate.md`).
- [COPAAA-19](/COPAAA/issues/COPAAA-19) — per-skill README row-update rule (B4 ratified 2026-04-26).
- [COPAAA-83](/COPAAA/issues/COPAAA-83) — evidence-package frontmatter-key enumeration (Rule C) and verbatim diff (Rule D).
