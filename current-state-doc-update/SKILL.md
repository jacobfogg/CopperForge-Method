---
name: current-state-doc-update
description: >
  Update CopperForge current-state documentation in place — README sections,
  per-skill `SKILL.md` bodies, registration / versioning text, and project-level
  state docs — so the repo always describes shipped behavior in present tense,
  replaces stale prose rather than appending to it, deep-links to canonical
  sources instead of duplicating their content, and keeps revision history in
  git rather than in the doc body. Use this skill when a CopperForge change
  makes any current-state doc on the repo describe a state that is no longer
  true at `HEAD`.
---

# Current-State Doc Update

Use this skill when you are landing (or have just landed) a CopperForge change that makes a current-state doc on the repo describe behavior that is no longer true at `HEAD`. The skill produces in-place edits — never new sections, never appended history. The four anchoring rules — present tense, replace don't append, deep-link don't duplicate, no in-doc revision history — define done; the workflow below is how you get there.

## What counts as a current-state doc

A current-state doc is any markdown on the CopperForge repo whose body asserts "this is how the repo or skill works today." That includes:

- `README.md` — project overview, registration matrix, skill catalog, versioning policy excerpts.
- Each skill's `SKILL.md` body and any `references/*.md` files — these document current skill behavior.
- Any project-level state doc that exists today or lands later (e.g., a registration matrix, a smoke-test summary).

The following are **not** current-state docs and are out of scope:

- `CHANGELOG.md` — a dated record, never edited retroactively.
- Issue descriptions, gate-review documents, comments — dated record.
- Commit messages and `git log` output — authoritative for change history.

## Preconditions

- The change is committed (or staged for commit) on a branch you can name `head`.
- You can list the behaviors that changed — typically from the gate-review or [evidence-package Artifact D](/COPAAA/issues/COPAAA-12#document-gate-review).
- For any new fact you might restate inline, you have located the canonical source (e.g., [versioning-policy §4.1](/COPAAA/issues/COPAAA-6#document-versioning-policy), [research-summary §3.1](/COPAAA/issues/COPAAA-2#document-research-summary)) so you can deep-link instead of duplicating.

## Workflow

1. **Inventory candidate docs.** From the change's diff, list every shipped doc whose assertions could now be wrong. Start with `README.md`, the changed skill's `SKILL.md` and its `references/*.md`, and any project-level state doc on `main`. Anything that asserts a fact about the repo, the skill catalog, registration, or versioning is a candidate.

2. **Triage each candidate.** Read the doc cold and ask: "Does every assertion still hold at `HEAD`?" Sort each section into one of:
   - **Still true** — leave untouched.
   - **Now wrong** — edit in place.
   - **Now redundant with a canonical source** — replace the local restatement with a deep link.

3. **Replace, do not append.** When a section is now wrong, rewrite the existing prose in place. Do NOT add a new section under a heading like "Update 2026-04-26" or "Recent changes". Do NOT keep the old assertion alongside the new one. The doc must read as if the new state was always the only state.

4. **Deep-link, do not duplicate.** When a fact has a canonical source, link to it: `[<short label>](/COPAAA/issues/COPAAA-X#document-<key>)` for issue documents, or the equivalent file path for repo files. Restating canonical facts inline is how docs drift; the link is how they stay correct. If the canonical source has not yet committed, treat its landing as a precondition for this update.

5. **Strip in-doc revision history.** If the doc contains a "Last updated", "Changelog", "History", "Authors", or version-table section inside its body, delete it as part of this update. Git history (`git log`, `git blame`) is the authoritative record. `CHANGELOG.md` is exempt — it IS the dated record, not a current-state doc.

6. **Rewrite to present tense.** Every assertion describes `HEAD`. "The importer accepts four source formats." "The repo contains seven skills." "The skill is registered as `jacobfogg/CopperForge-Skills/<name>`." Reject "was added", "will be", "currently being implemented", "recently", "now supports", or "as of `<date>`" — those phrasings either belong in `CHANGELOG.md` / git history, or signal a doc that should be updated when the underlying work lands.

7. **Cite the change in the commit message, not the doc.** The commit message names the task that drove the doc update (e.g., `current-state-doc-update: README registration matrix (COPAAA-13)`). The doc body itself stays free of cross-references to the task that updated it.

8. **Self-check.** Read each edited section as a cold reader. If you can detect that a recent change happened — phrasing like "now", "new", "recently", "no longer" — rewrite. The reader should not be able to tell when the section was last edited from the prose alone. Then run `git diff <base> -- <doc-path>` and confirm the diff is small and surgical; a whole-file rewrite is a smell that usually means the doc was append-history before.

## Quality bar

- **Present tense, no tense leakage.** Reject "was", "will be", "currently being", "in progress", "recently", "as of `<date>`".
- **Replace, never append.** No "Update YYYY-MM-DD" sections, no parallel old/new claims, no `<!-- TBD -->` markers left behind once the underlying work has landed.
- **Deep-link, never duplicate canonical facts.** If [versioning-policy](/COPAAA/issues/COPAAA-6#document-versioning-policy) defines the bump rules, the README links to it; the README does not paraphrase §4.1.
- **No in-doc revision history.** No "Last updated", "Changelog", "Authors", or version tables inside a current-state doc. (`CHANGELOG.md` itself is exempt.)
- **Diffs are small and surgical.** A current-state doc update typically touches one or two paragraphs. A whole-file rewrite usually means the doc was append-history before; cite the underlying drift in the commit message and proceed.
- **Auditable from `git diff`.** Anyone running `git diff <base> <head> -- <doc-path>` can map the change to the underlying behavior change without reading the doc body's prose history.

## Out of scope

- Editing issue descriptions, gate-review documents, comments, or any other dated record.
- Adding new current-state docs not driven by the change. If a new doc is needed, that is a separate task.
- Performing a full repo doc audit. The skill is per-task; scope is the docs the current change made wrong, not the full repo. Cross-change reconciliation lives in [doc-reconciliation](/COPAAA/issues/COPAAA-18) at epic close.
