---
name: copperforge-reporter
description: >
  Methodology-aware reporter for CopperForge. Driven by Paperclip routines that
  spawn an execution issue assigned to the reporter agent each fire; this skill
  is the heartbeat-time entry point that detects which trigger fired and
  dispatches to the correct trigger-reference under `references/`. v1 ships
  the **blocker-trigger only** (real-time alert when any issue in the agent's
  company transitions to `blocked` status), per [COPAAA-104](/COPAAA/issues/COPAAA-104)
  vertical-slice anchor scope. The digest-trigger and quiet-stall-trigger are
  authored in B4 / B5 and added under `references/` without changing this
  SKILL.md body. The reporter calls `slack-transport` for delivery and owns
  zero composition logic itself — composition lives in
  `references/slack-payload-shapes.md`. Use this skill when the reporter agent
  picks up a routine-spawned execution issue and needs to decide what to read,
  what to compose, and what to deliver.
---

# copperforge-reporter

Use this skill when the reporter agent picks up a routine-spawned execution issue (the routine title starts with `[reporter]` and the issue's `originKind` is `routine`). This skill is the heartbeat-time entry point: it reads the spawned issue's context, identifies which trigger fired, and dispatches to the matching reference under `references/`. The reporter never composes Slack content directly in this body — `references/slack-payload-shapes.md` owns block-kit shapes and `references/blocker-trigger.md` owns the v1 detection algorithm. The reporter never delivers Slack content directly either — `slack-transport` ([COPAAA-103](/COPAAA/issues/COPAAA-103)) is the only outbound path. This skill is intentionally narrow on the SKILL.md surface (dispatch + heartbeat invariants) so each trigger reference is self-contained and the per-trigger surfaces can grow under B4 (digest) and B5 (quiet-stall) without re-touching this file. Per the COPAAA-61 board direction, the reporter is the only place CopperForge methodology awareness lives in the delivery pipeline; `slack-transport` carries no methodology and any caller can use it.

## Preconditions

- The routine that spawned the execution issue is registered in the reporter agent's company per [`paperclip/references/routines.md`](https://github.com/jacobfogg/CopperForge-Method/blob/main/methodology/) lifecycle. v1 registers exactly one routine — the blocker-trigger schedule trigger — per `references/blocker-trigger.md`. Routine registration is a one-time CEO step (`agents:create` permission, per [skill_import_permission.md](https://github.com/jacobfogg/CopperForge-Method/blob/main/methodology/) and BA research-summary §5.3); it is NOT performed inside this skill body.
- The reporter agent identity is the agent the routine fires as. Per [COPAAA-85 plan §4 OQ-6](/COPAAA/issues/COPAAA-85#document-plan), the reporter-agent identity is resolved by board approval [`e58daa4d-…`](/COPAAA/approvals/e58daa4d-7f75-4d52-917f-4f6741f25f4d) ([COPAAA-102](/COPAAA/issues/COPAAA-102)) — either (a) a new dedicated `reporter` agent or (b) a CTO-owned fallback. This skill is parametric on the agent identity; the SKILL.md body does NOT name a specific agent. The Code Gate evidence package binds this skill to whichever path the board ratifies.
- `slack-transport` is registered in the same company at the version pinned by the CopperForge-Method commit SHA. The reporter invokes `slack-transport` by name and passes a `message` argument matching the slack-transport input contract ([COPAAA-103 SKILL.md preconditions](/COPAAA/issues/COPAAA-103)). The reporter does NOT call Slack webhooks directly.
- A Paperclip company-level `secret_ref` binding populates an environment variable named per the OQ-5 production binding decision (default `SLACK_WEBHOOK_URL` per [COPAAA-103 OQ-5 ack](/COPAAA/issues/COPAAA-103)) at heartbeat-launch. The reporter passes the env-var name to `slack-transport` via the `webhookEnvVar` argument; the URL itself is read by `slack-transport`, never by this skill.
- The reporter agent has read access to `/api/companies/:companyId/issues`, `/api/companies/:companyId/activity`, `/api/issues/:id/activity`, `/api/issues/:id/comments`, and `/api/companies/:companyId/heartbeat-runs` (BA research-summary §2.1, §2.2, §2.3, §2.4). Read access to `/api/agents/:id/runtime-state` is NOT required and is Board-gated for non-Board agents (BA §2.4); this skill never calls that endpoint.
- This skill makes zero outbound network calls except (a) Paperclip API reads via `Authorization: Bearer $PAPERCLIP_API_KEY` (BA §2.6), (b) Paperclip API writes (idempotence-sentinel comments and execution-issue summary comments) via the same auth plus `X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID`, and (c) `slack-transport` invocations. Direct outbound Slack POSTs from this skill are gate-failing on the quality bar below.

## Workflow (heartbeat procedure)

1. **Identify spawned-issue context.** Read `PAPERCLIP_TASK_ID` from the environment (the execution issue id assigned by the routine engine). `GET /api/issues/{PAPERCLIP_TASK_ID}` and read `originKind`, `originId`, `title`, and `description`. If `originKind !== "routine"`, the spawn was not from a routine — log a no-op summary on the execution issue (`[reporter] no-op: originKind={value}`) and exit cleanly. The reporter does NOT pick up manual or board-spawned work; routine-spawned issues are its only input.

2. **Resolve trigger kind.** The routine title prefix encodes the trigger kind. The v1 convention (set at routine registration time in `references/blocker-trigger.md`) is `[reporter:blocker-trigger] <human-readable label>`. Read the title prefix from `[reporter:` to the next `]`; the substring between is the trigger key. Allowed v1 keys (this list grows in B4 / B5):
   - `blocker-trigger` — dispatch to `references/blocker-trigger.md`.
   - Any other key (including `digest-trigger` and `quiet-stall-trigger`) — log `[reporter] unknown trigger key: {key} — not yet shipped at this commit SHA`, post a no-op summary on the execution issue, and exit cleanly. v1 ships only blocker-trigger; B4 / B5 add the others under their own per-skill Code Gates without touching this file.
   - Title prefix missing or malformed (no `[reporter:...]` brackets) — log `[reporter] malformed routine title: {title}` and exit cleanly. The malformed-title path is non-throwing because operator misconfiguration of routine titles must NOT crash the heartbeat — exit with a summary so the operator sees the message in the routine-spawned issue.

3. **Dispatch to trigger reference.** Load the trigger reference file by exact path under `references/` (per the matrix in step 2). Each trigger reference owns its own end-to-end logic:
   - The detection or scheduling rule (when does this trigger fire? what does it read?).
   - The composition rule (what shape does the Slack message take? — typically by referring to `references/slack-payload-shapes.md`).
   - The idempotence rule (what state does it consult to avoid duplicate alerts?).
   - The result envelope (what does it report back to step 4 of this workflow?).
   The reference returns either `{ ok: true, alertsSent: N, cursor: <opaque> }` (success) or `{ ok: false, error: <string>, partialAlertsSent: N }` (partial / failure). Step 4 below handles both shapes uniformly.

4. **Record fire outcome on the execution issue.** Post one comment on `PAPERCLIP_TASK_ID` summarizing the fire — alerts sent, cursor advanced, partial-failure context. The literal phrase `[reporter:{trigger-key}] fire-summary` MUST appear at the top of the comment so operators can `grep` for fire-summaries across the routine-spawned issue history. The comment body includes:
   - `runId`: `$PAPERCLIP_RUN_ID`.
   - `triggerKey`: the value resolved in step 2.
   - `alertsSent`: integer count from the trigger reference's result envelope.
   - `cursor` (if applicable): the opaque cursor value the trigger reference emitted (e.g., last-processed activity event id for blocker-trigger).
   - `error` (if applicable): the literal error string from the trigger reference's result envelope; absent if `ok: true`.
   This comment is the per-fire audit trail; together with the per-issue idempotence-sentinel comments emitted by `references/blocker-trigger.md` (and equivalents in B4 / B5), it forms the reporter's complete observability surface inside Paperclip.

5. **Exit the heartbeat.** Mark the execution issue `done` via `PATCH /api/issues/{PAPERCLIP_TASK_ID}` with `status: "done"` (and `X-Paperclip-Run-Id`). The routine engine creates the next execution issue on the next trigger fire. The reporter does NOT mark the execution issue `blocked` on partial failure — partial failure is recorded in the step-4 comment with `error`, but the issue still completes so the routine concurrency policy (`coalesce_if_active` / `skip_if_active` / `always_enqueue` per [`paperclip/references/routines.md`](https://github.com/jacobfogg/CopperForge-Method/blob/main/methodology/)) advances normally. Operators reading `[reporter:{trigger-key}] fire-summary` comments with non-empty `error` fields drive the remediation; the reporter does not self-block.

## Quality bar

- Zero Slack composition logic in this SKILL.md body. Block-kit shapes, header text, mrkdwn escaping, per-block character limits, and section-truncation rules all live in `references/slack-payload-shapes.md`. Adding any of those to this file is gate-failing on the body-narrowness rule (mirrors the `slack-transport` board direction "Resist the temptation to merge slack-transport and copperforge-reporter").
- Zero direct Slack webhook POSTs from this body. The only outbound delivery path is `slack-transport`. A code path that POSTs directly to a Slack webhook URL — including for "diagnostic" or "smoke" purposes — is gate-failing on T-direct-post in the QA2 test rubric.
- Trigger references are loaded only when their trigger key matches step 2's resolution. The SKILL.md body does NOT pre-load `references/blocker-trigger.md` or any other reference at heartbeat start; references are read on demand per the Paperclip canonical pattern (`paperclip/references/routines.md` line 187 cited in BA research-summary §4: *"the SKILL.md body is always loaded; everything in `references/` is read on demand by the agent when the SKILL.md instructs it to"*). This keeps the always-loaded surface small and bounds the per-fire context cost.
- Idempotence is enforced per-trigger (in the trigger reference), not in the SKILL.md body. The body does NOT consult any cursor, run-history, or per-issue sentinel comment — that responsibility belongs to the trigger reference because each trigger has different idempotence semantics (blocker-trigger uses per-(issueId, activityId) sentinels; digest-trigger uses cron-fire windows; quiet-stall-trigger uses agent-finishedAt timestamps). Pulling idempotence into the body collapses the future expansion surface.
- The literal phrases QA2 pattern-matches in T1–T4 of the per-skill Test Gate rubric MUST appear verbatim in this body:
  - T1 (originKind not routine): `[reporter] no-op: originKind=`
  - T2 (unknown trigger key): `[reporter] unknown trigger key:`
  - T3 (malformed title): `[reporter] malformed routine title:`
  - T4 (fire-summary): `[reporter:{trigger-key}] fire-summary` (the literal `{trigger-key}` is a substitution token; QA2's regex matches the prefix `[reporter:` and the suffix `] fire-summary` with any non-empty key in between).
- The reporter does NOT exit the heartbeat with status `error` or `blocked` on partial Slack delivery failure (e.g., `slack-transport` returns `{ ok: false, error: "retries_exhausted", attempts: 4 }`). Partial failure is recorded in the step-4 fire-summary comment with the `error` field populated; the execution issue still transitions to `done`. Self-blocking on Slack delivery would create a feedback loop: a Slack outage would block the reporter, the reporter is the system that alerts on blockers, and the board would lose blocker visibility precisely when delivery is broken. Recording-and-completing keeps the audit trail intact and the routine cadence steady.
- The reporter does NOT call `/api/agents/:id/runtime-state` (Board-gated per BA §2.4). Adding that call is gate-failing on the unauthorized-endpoint rule. Quiet-stall detection (B5) reads `/api/companies/:companyId/heartbeat-runs` instead.
- The reporter does NOT call `/api/companies/:companyId/secrets` to resolve the webhook URL. The webhook URL arrives via env-var injection mediated by `slack-transport` (BA §5.1, [COPAAA-103](/COPAAA/issues/COPAAA-103)). Direct secret-API access is gate-failing on the secrets-injection rule (and would also 403 for non-Board agents per BA §2.4 / §5.1 live-test).
- Per-skill README row update is mandatory at Code Gate per the [per_skill_readme_row_rule.md](https://github.com/jacobfogg/CopperForge-Method/blob/main/methodology/) rule (COPAAA-19 B4 ratified). The `copperforge-reporter` row in repo `README.md` lands in the same coding commit as this SKILL.md and is reaffirmed in the Code Gate evidence package.
- Frontmatter contains `name` and `description` only — no `model`, `allowed-tools`, or input schema. This mirrors the four canonical Paperclip alpha skills (BA research-summary §4: *"all four use only `name` and `description`"*).

## References

- [`references/blocker-trigger.md`](references/blocker-trigger.md) — v1 trigger: scans `/api/companies/:id/issues?status=blocked` and `/api/companies/:id/activity` for `action: "issue.updated"` records with `details.status === "blocked"` and `details._previousStatus !== "blocked"` (BA §2.2). Routine registration shape, detection algorithm, idempotence sentinel format, F1 `?status` family live-smoke wiring per [COPAAA-104 §F1](/COPAAA/issues/COPAAA-104).
- [`references/slack-payload-shapes.md`](references/slack-payload-shapes.md) — block-kit shapes for Slack messages emitted by the reporter. v1 ships the `blocker-alert` shape (used by `blocker-trigger`); B4 / B5 add `digest-summary` and `quiet-stall-alert` shapes here without touching this SKILL.md. Per-block character limits cited from BA research-summary §1.3 (Assumption — re-verified at this skill's Code Gate per OQ-2 Dev re-verification).

## v1 scope summary (what does and does not ship at this commit SHA)

This skill body and the two `references/` files referenced above are the entirety of the v1 ship. The blocker-trigger routine registration is documented in `references/blocker-trigger.md` and is performed by CEO at registration time per BA §5.3 (`agents:create` permission). Specifically NOT shipped at v1 (deferred per [COPAAA-85 plan §B4 / §B5](/COPAAA/issues/COPAAA-85#document-plan) and the [COPAAA-104 issue scope](/COPAAA/issues/COPAAA-104)):

- Digest trigger (twice-daily 6 AM / 6 PM America/Denver schedule) — deferred to **B4** ([COPAAA-106](/COPAAA/issues/COPAAA-106)). B4 adds `references/digest-trigger.md` and registers the second routine.
- Quiet-stall trigger (15-min cadence on `[queue-status]` comments) — deferred to **B5** ([COPAAA-108](/COPAAA/issues/COPAAA-108)). B5 adds `references/quiet-stall-trigger.md` and registers the third routine, after `queue-status` data has accumulated for ≥4 hours per the parent issue's vertical-slice ordering.
- `queue-status` invocation hook — that is **B6** ([COPAAA-107](/COPAAA/issues/COPAAA-107)) and a separate skill (`queue-status`) entirely; it does not ride on this skill.

The shape of step 2's dispatch table is intentionally additive — the unknown-trigger-key branch returns cleanly without erroring, so adding `digest-trigger` and `quiet-stall-trigger` rows in B4 / B5 is a one-line edit per row plus a new reference file. No SKILL.md frontmatter or workflow-step renumbering is required to ship those triggers.
