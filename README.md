# CopperForge Method

CopperForge is the agent company that builds and maintains the methodology skill library used across CopperPress. This repo is the source of truth for those skills.

## Skills (v1)

The v1 release ships seven skills, in build-dependency order:

1. `evidence-package` — packages per-gate evidence in a structured, auditable form
2. `current-state-doc-update` — updates project current-state docs alongside code changes
3. `auditor-challenge` — structured challenges produced by the Auditor at every gate
4. `research-summary` — BA-facing structure for Plan-Gate research summaries
5. `gate-review` — QAM-facing structure for running and recording Test/Code/Epic gates
6. `test-change-request` — Dev → QAM mid-epic communication for changing test scope
7. `doc-reconciliation` — QA-facing pass at epic close to reconcile docs against shipped code

Each skill lives in its own top-level directory. Per-skill `README.md` and `SKILL.md` content lands during that skill's vertical slice.

## Operator skills

Operator skills are CEO-only and act on a target Paperclip company's installation state rather than on a per-gate artifact. They are versioned and shipped on the same cadence as the per-role methodology skills above.

| Skill | Supported operations | Parent issue |
|---|---|---|
| `methodology-bootstrap` | `install` | [COPAAA-58](/COPAAA/issues/COPAAA-58) |

The `upgrade` (board-approval gated) and `drift-check` (read-only) operations of `methodology-bootstrap` accumulate onto this row as they land via [COPAAA-69](/COPAAA/issues/COPAAA-69) and [COPAAA-79](/COPAAA/issues/COPAAA-79) respectively, per the COPAAA-58 plan §4 C4 single-row convention.

## Versioning

This repository follows [Semantic Versioning](https://semver.org/). `v1.0.0` is the first published release once all seven skills have passed their per-skill Code Gate and the epic-close gate has signed off. Tags are protected — see `v*` tag-protection on the repository settings.

## License

[MIT](LICENSE).
