# copperforge-reporter — blocker-trigger reference (v1)

This reference is loaded on demand by `SKILL.md` step 3 when the routine title's trigger key is `blocker-trigger`. It owns the v1 vertical-slice end-to-end logic: routine registration shape, detection algorithm, idempotence rule, and result envelope. Keep this file self-contained — `SKILL.md` does not duplicate any of this content, and B4 / B5 trigger references mirror this shape independently.

## Routine registration

Registered once by CEO at company-init time per BA [research-summary §5.3](/COPAAA/issues/COPAAA-62#document-research-summary) (the `/skills/import` family of endpoints requires `agents:create`, which only CEO holds — skill-library mutations are CEO-only per [COPAAA-58](/COPAAA/issues/COPAAA-58)). Registration is NOT performed by this skill at heartbeat time — it is a one-time CEO step recorded in the Code Gate evidence package per [COPAAA-104](/COPAAA/issues/COPAAA-104).

```
POST /api/companies/{companyId}/routines
{
  "title": "[reporter:blocker-trigger] real-time blocker alerts",
  "description": "Scans for issue.updated activity records where details.status === 'blocked' and details._previousStatus !== 'blocked', delivers a Slack alert per fresh transition.",
  "assigneeAgentId": "{reporter-agent-id}",
  "projectId": "{copperforge-project-id}",
  "priority": "high",
  "status": "active",
  "concurrencyPolicy": "skip_if_active",
  "catchUpPolicy": "skip_missed"
}
```

Then add the schedule trigger:

```
POST /api/routines/{routineId}/triggers
{
  "kind": "schedule",
  "cronExpression": "* * * * *",
  "timezone": "UTC",
  "label": "every-minute"
}
```

**Key registration choices and reasoning:**

- `concurrencyPolicy: skip_if_active` — if the previous one-minute fire is still working through a backlog (e.g., a wave of 30 issues transitioning to `blocked` simultaneously), the next fire is skipped rather than coalesced. Skip is preferred over coalesce because each fire is itself complete (every fire reads the full activity window since cutoff and emits all alerts up to the cutoff; a coalesced fire would re-read the same window and risk re-alerting on the same activity events if the sentinel-comment write lagged the next fire's start).
- `catchUpPolicy: skip_missed` — missed fires (server downtime) are dropped. This is correct because the next fire's cutoff (see step 1 of the algorithm below) is the reporter's last successful `finishedAt`, which naturally widens to cover the downtime — so no missed-fire enqueue is needed; the next live fire processes the entire downtime window in one pass.
- `cronExpression: "* * * * *"` (every minute) — "real-time" per the parent issue's wording is satisfied by a one-minute polling cadence. Tighter cadences (e.g. every 30 seconds) are not supported by standard 5-field cron syntax (`paperclip/references/routines.md` line 102: *"`cronExpression`: standard 5-field cron syntax"*).
- `timezone: "UTC"` — the blocker-trigger is realtime, not DST-sensitive (cron fires every minute regardless of timezone). UTC is chosen so the registration is timezone-stable; the digest-trigger (B4) uses `America/Denver` for human-aligned 6 AM / 6 PM cadence.
- The routine title prefix `[reporter:blocker-trigger]` is the literal string `SKILL.md` step 2 pattern-matches. Changing this prefix without updating `SKILL.md` step 2 will cause the heartbeat to dispatch to the unknown-trigger-key branch and exit cleanly with no alerts emitted.

## Skill input parameters

The trigger reference reads the following inputs from the spawned execution issue's skill-input bag (per Paperclip routine-spawned-issue convention — inputs declared at routine registration are surfaced into each fire's execution issue):

| Input | Default | Notes |
|---|---|---|
| `webhookEnvVar` | `SLACK_WEBHOOK_URL` | Passed verbatim to `slack-transport`. The OQ-5 production binding name lives here; v1 single-webhook ack from CEO sets this default per [COPAAA-103 OQ-5](/COPAAA/issues/COPAAA-103). |
| `slackChannelOverride` | `null` | Reserved for future multi-channel work; v1 single-webhook ignores this. |
| `cutoffFallbackMinutes` | `5` | Used for first-ever run when `heartbeat-runs` returns no prior `finishedAt` for the reporter agent. Five minutes is short enough to bound the cold-start fan-out and long enough to absorb routine-engine startup jitter. |

Inputs missing from the skill-input bag use the defaults above.

## Detection algorithm

The algorithm is purely API reads up to step 6, then a single `slack-transport` call per fresh transition, then a single sentinel-comment write per delivered alert. Every step has an explicit failure-mode that short-circuits cleanly (no exceptions propagate to `SKILL.md` step 4 — the trigger reference catches them and returns a structured result envelope).

1. **Resolve cutoff timestamp.** Read the reporter agent's most recent finished run from `GET /api/companies/{companyId}/heartbeat-runs?agentId={reporterAgentId}&limit=1`. Filter the response array for entries with `status: "finished"` (NOT `error`, NOT `running`). If at least one finished run exists, set `cutoff = entry.finishedAt`. Otherwise (cold start), set `cutoff = (now - cutoffFallbackMinutes minutes)`. The `now` reference is the heartbeat start time from `Date.now()`. The cutoff is exclusive: only activity events with `createdAt > cutoff` are considered fresh.

2. **Self-check the `?status` multi-status filter (F1 wiring).** Issue `GET /api/companies/{companyId}/issues?status=todo,in_progress,blocked&limit=1` and verify HTTP 200 with an array body. This call is included on every fire to satisfy [COPAAA-104 §F1](/COPAAA/issues/COPAAA-104) Code Gate evidence — *"`?status=todo,in_progress,blocked` (multi-status filter — proves the param parses end-to-end)"*. The response body is not consumed by the algorithm; the call exists solely to assert the F1 wiring resolves at the API edge on every fire (so a regression in the multi-status parser is caught at the next routine fire instead of at digest-trigger time in B4). On non-200 response, abort the fire with `{ ok: false, error: "f1_status_filter_self_check_failed", partialAlertsSent: 0 }` — the reporter prefers a hard abort here because a broken status-filter parser corrupts blocker detection downstream.

3. **Fetch current blockers (F1 `?status=blocked` consumed).** Issue `GET /api/companies/{companyId}/issues?status=blocked&limit=200` (200 is the page size; if the company has >200 blocked issues at any one time, the methodology has a deeper problem than the reporter and a single page misses none of the active blockers in any realistic CopperForge state). Save the response array as `currentBlockers` for use in step 6 cross-reference. The `?status=blocked` call is the second F1 evidence point — every fire writes both to the audit trail.

4. **Fetch activity since cutoff.** Issue `GET /api/companies/{companyId}/activity?after={cutoff}&limit=500`. Filter the response array client-side to entries matching all three predicates:
   - `action === "issue.updated"`.
   - `details.status === "blocked"`.
   - `details._previousStatus !== "blocked"` (the canonical "fresh transition into blocked" signal per BA §2.2; explicitly excludes `blocked → blocked` no-op updates).
   Sort the filtered array by `createdAt` ascending (oldest first) so alerts emit in chronological order — this preserves operator readability when multiple issues transition in the same fire. Save the filtered + sorted array as `freshTransitions`.

5. **Per-fresh-transition idempotence check.** For each entry in `freshTransitions`, fetch the comments on the corresponding issue (`entityId` field on the activity record): `GET /api/issues/{entityId}/comments?limit=200`. Pattern-match each comment body against the literal regex `^\[reporter:blocker-alert\] activityId={activityId} `. If at least one match exists, the alert was already delivered for this specific (issueId, activityId) pair on a prior fire — skip this entry (do NOT emit a duplicate alert; do NOT re-write the sentinel). The literal phrase `[reporter:blocker-alert] activityId=` MUST appear so QA2's T-idempotence pattern-match works. The `activityId` token is the activity record's `id` field — globally unique per Paperclip; the per-(issueId, activityId) sentinel guarantees that a re-fire of the same status transition produces no duplicate alert, while a fresh transition (different `activityId`, even on the same issue) produces a fresh alert per the parent issue's "If the same issue cycles back into `blocked` after being unblocked, the alert re-fires with the new context" requirement.

6. **Compose and deliver.** For each fresh, non-idempotent transition (passed steps 4 and 5):
   - Resolve the issue's full record (use the matching entry from `currentBlockers` if present; otherwise `GET /api/issues/{entityId}` for completeness — covers the rare race where an issue transitioned to `blocked` then immediately changed status before the activity poll, in which case it's NOT in `currentBlockers` but the alert still fires for the historical transition).
   - Read the latest comment on the issue (`GET /api/issues/{entityId}/comments?limit=1&order=desc`) — this is the "blocker context" comment per the parent description's "what specifically is blocked (read from the blocking comment)". If no comments exist, the alert renders the issue's `description` truncated to fit the per-block character budget (cited from BA §1.3 — re-verified at Code Gate per OQ-2).
   - Compose the Slack block-kit message body using the `blocker-alert` shape from `references/slack-payload-shapes.md`. The shape consumes: issue identifier (e.g., `COPAAA-104`), title, status, priority, assignee agent name, blocker-context excerpt, and a deep link of the form `https://{paperclip-instance-host}/issues/{identifier}`.
   - Invoke `slack-transport` with `{ message: <composed body>, webhookEnvVar: <input.webhookEnvVar> }`. Capture the `slack-transport` result envelope (`{ ok, status, attempts, error?, retryAfter?, note? }` per [COPAAA-103 SKILL.md §5](/COPAAA/issues/COPAAA-103)).
   - On `ok: true`, post the idempotence sentinel: `POST /api/issues/{entityId}/comments` with body `[reporter:blocker-alert] activityId={activityId} runId={PAPERCLIP_RUN_ID} status=ok attempts={attempts}`. The sentinel is written AFTER the Slack delivery succeeds, NOT before — writing the sentinel before the POST would risk a missed alert (sentinel exists but Slack failed). The chosen ordering (Slack-first, sentinel-second) risks at most a duplicate alert on a `slack-transport`-success-but-sentinel-write-fails race; this is preferred over a missed-alert race because the cost-of-error is "operator sees two notifications" instead of "operator misses a blocker."
   - On `ok: false` (retries exhausted, permanent 4xx, etc.), do NOT write the idempotence sentinel — leave the activity record un-marked so the next fire's step 5 will retry the alert. Record the failure in the result envelope's `partialAlertsSent` count (incremented for delivered alerts only — failed-and-not-sentineled transitions are NOT counted as alertsSent).

7. **Emit the result envelope.** The trigger reference returns to `SKILL.md` step 4 with one of two shapes:
   - Success: `{ ok: true, alertsSent: <integer count of delivered + sentineled alerts>, cursor: <largest activity.createdAt processed in this fire> }`.
   - Partial / failure: `{ ok: false, error: <literal string describing the failure mode>, partialAlertsSent: <integer count of delivered alerts before failure>, cursor: <largest activity.createdAt successfully processed before failure> }`.
   Both shapes carry `cursor` because `SKILL.md` step 4's fire-summary comment surfaces it for operator inspection. The cursor is informational only — the cutoff resolution in step 1 does NOT consult the previous fire's emitted cursor (it consults `heartbeat-runs.finishedAt`); this avoids a single-point-of-failure if a sentinel-comment write fails.

## Idempotence sentinel format (canonical)

The idempotence sentinel is a comment posted on the **blocked issue** (NOT on the routine-spawned execution issue) with the literal body shape:

```
[reporter:blocker-alert] activityId={activityId} runId={runId} status=ok attempts={attempts}
```

Where:
- `{activityId}` is the activity record's `id` field (UUID).
- `{runId}` is the value of `PAPERCLIP_RUN_ID` for the heartbeat that delivered the alert.
- `{attempts}` is the `attempts` field from the `slack-transport` result envelope (1, 2, 3, or 4).
- `status=ok` is hardcoded — failed deliveries do NOT write a sentinel (see step 6 ordering).

Step 5's pattern match keys on the prefix `[reporter:blocker-alert] activityId={activityId} ` (with a trailing space). The trailing space prevents prefix-collision with future sentinel formats that might extend the activityId field with a dot or hyphen suffix. Operators reading these comments see the full audit trail per blocked issue: every transition into `blocked` yields one sentinel comment, every re-transition yields a fresh sentinel with a different `activityId`.

## F1 `?status` family wiring (per [COPAAA-104](/COPAAA/issues/COPAAA-104) Code Gate evidence)

This trigger consumes both `?status` filter shapes documented in BA research-summary §2.1 and required as live-smoke evidence at the per-skill Code Gate per [COPAAA-85 plan §B2 F1](/COPAAA/issues/COPAAA-85#document-plan):

- **`?status=blocked`** — algorithm step 3, every fire. Code Gate evidence captures one fire's HTTP response showing the `?status=blocked` parameter parsed correctly and an array of blocked-status issues returned.
- **`?status=todo,in_progress,blocked`** — algorithm step 2, every fire. Code Gate evidence captures one fire's HTTP response showing the multi-status comma-list parameter parsed correctly and an array (possibly empty) returned. The fact that the array is observably non-empty on a real CopperForge company at any active day-of-work moment satisfies the "param parses end-to-end" assertion — the parser would short-circuit on a malformed comma-list with HTTP 400, not return HTTP 200 with an empty array.

The other two F1 params (`?parentId=` and `?participantAgentId=`) are NOT consumed by this trigger and are smoked at B4 / B5 respectively per the plan's split. A reviewer who finds those params in this reference is reading a stale draft; v1 ships only `?status` evidence.

## Failure modes and result-envelope error strings (canonical list)

The trigger reference returns one of these literal `error` strings on `ok: false` so QA2 can pattern-match in T5–T8 of the per-skill Test Gate rubric:

- `f1_status_filter_self_check_failed` — step 2 returned non-200 (multi-status parser regression).
- `activity_fetch_failed` — step 4 returned non-200.
- `current_blockers_fetch_failed` — step 3 returned non-200.
- `cutoff_resolution_failed` — step 1 returned non-200 from `/heartbeat-runs`.
- `slack_transport_unrecoverable` — step 6 returned `ok: false` from `slack-transport` with no retry budget remaining (per [COPAAA-103 SKILL.md §4](/COPAAA/issues/COPAAA-103) the `error` field on the slack-transport envelope carries the underlying reason, e.g., `retries_exhausted` or a 4xx named error; this trigger's envelope wraps that inside `slack_transport_unrecoverable` for a stable upstream surface). The `partialAlertsSent` field counts deliveries completed before the failure.
- `idempotence_check_failed` — step 5 returned non-200 from the per-issue comments fetch.
- `sentinel_write_failed` — step 6 returned non-200 from the sentinel-comment POST. The Slack alert was already delivered when this happens; the next fire will re-deliver because the sentinel is missing — the alert-once invariant is preserved at the cost of one duplicate per failure (acceptable per the step-6 ordering reasoning above).

These error strings are the trigger reference's stable contract with the SKILL.md fire-summary comment writer; renaming them collapses the QA2 test surface and is gate-failing on the canonical-list rule above.
