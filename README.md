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

## Reporter skill set

Reporter skills are the methodology's self-defense layer — they surface blockers, digests, and quiet-stall signals to the board on a timely cadence so the methodology cannot silently fall behind. They live under the `skills/` subdirectory (separate from the v1 per-role methodology skills above) and are built and shipped under [COPAAA-61](/COPAAA/issues/COPAAA-61).

| Skill | Role | Parent issue |
|---|---|---|
| `slack-transport` | Pure delivery primitive — single block-kit POST to one Slack incoming-webhook URL with a four-class retry policy (200 ok / 4xx named-error / 429 Retry-After / 5xx backoff, capped at 3 retries). No CopperForge knowledge. | [COPAAA-103](/COPAAA/issues/COPAAA-103) |
| `copperforge-reporter` | Methodology-aware reporter dispatched by Paperclip routines. v1 ships the **blocker-trigger** only (real-time alert when any issue transitions to `blocked`). Calls `slack-transport` for delivery. References under `skills/copperforge-reporter/references/` add `digest-trigger` (B4) and `quiet-stall-trigger` (B5) without changing the SKILL.md body. | [COPAAA-104](/COPAAA/issues/COPAAA-104) |
| `queue-status` | Heartbeat-time self-introspection primitive. Reads `GET /api/agents/me/inbox-lite` and posts one structured `[queue-status]` comment on the agent's currently-claimed execution issue (`PAPERCLIP_TASK_ID`). The line format is the parser contract that the `copperforge-reporter` quiet-stall trigger (B5) depends on — see `skills/queue-status/references/output-schema.md`. No methodology, no Slack delivery. | [COPAAA-105](/COPAAA/issues/COPAAA-105) |

## Versioning

This repository follows [Semantic Versioning](https://semver.org/). `v1.0.0` is the first published release once all seven skills have passed their per-skill Code Gate and the epic-close gate has signed off. Tags are protected — see `v*` tag-protection on the repository settings.

## License

[MIT](LICENSE).
