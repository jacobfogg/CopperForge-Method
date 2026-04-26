# copperforge-reporter — digest-trigger reference (v2)

This reference is loaded on demand by `SKILL.md` step 3 when the routine title's trigger key is `digest-trigger`. It owns the v2 vertical-slice end-to-end logic for the twice-daily per-parent project digest: routine registration shape, scheduling and parent-resolution rules, three-section composition pipeline, idempotence rule, and result envelope. The file is self-contained — `SKILL.md` does not duplicate any of this content, and `references/blocker-trigger.md` is unaffected.

## Routine registration

Registered once by CEO at company-init time per BA research-summary §5.3 (the `/skills/import` family of endpoints requires `agents:create`, which only CEO holds per [skill_import_permission.md](https://github.com/jacobfogg/CopperForge-Method/blob/main/methodology/)). Registration is NOT performed by this skill at heartbeat time — it is a one-time CEO step recorded in the Code Gate evidence package per [COPAAA-106](/COPAAA/issues/COPAAA-106).

```
POST /api/companies/{companyId}/routines
{
  "title": "[reporter:digest-trigger] twice-daily project digest",
  "description": "Twice-daily 6 AM + 6 PM (America/Denver) per-parent digest with three sections: Active Blockers, Accomplished Since Last Digest, Remaining Work. Reads /dashboard + /issues?parentId=... + /activity per BA research-summary §2.5.",
  "assigneeAgentId": "{reporter-agent-id}",
  "projectId": "{copperforge-project-id}",
  "priority": "medium",
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
  "cronExpression": "0 6,18 * * *",
  "timezone": "America/Denver",
  "label": "twice-daily-mountain"
}
```

**Key registration choices and reasoning:**

- `cronExpression: "0 6,18 * * *"` — fires at minute 0 of hours 6 and 18 every day in the configured timezone. The 6 AM / 6 PM cadence is the parent description's literal request (COPAAA-61 reporter epic). Standard 5-field cron syntax per `paperclip/references/routines.md` line 102 (cited in BA research-summary §3.2).
- `timezone: "America/Denver"` — chosen because the CopperForge operator (CEO) is in the Mountain timezone; "6 AM" and "6 PM" are wall-clock semantics, not UTC. Denver observes US DST (MST/MDT) which is the F3 correctness-proof case at Code Gate. The `Intl.DateTimeFormat`-backed `nextCronTickInTimeZone` primitive (`@paperclipai/server/dist/services/routines.js` line 76) handles DST transitions correctly via ICU's IANA tz database delegation — see Code Gate evidence-package §F3 for the verbatim function-body inspection proof.
- `concurrencyPolicy: skip_if_active` — if the previous fire is still working through a multi-parent fan-out (e.g., the company has six parent epics and one digest fire is mid-composition), the next fire is skipped. Twice-daily cadence makes this a cold-path concurrency policy — `skip_if_active` only fires meaningfully if a single fire takes >12 hours, which would itself be a methodology failure mode. Skip is preferred over coalesce so the next missed fire is recorded as a `routine_runs.status='skipped'` row (operator-visible) instead of silently merged.
- `catchUpPolicy: skip_missed` — missed fires (server downtime spanning a 6 AM or 6 PM tick) are dropped rather than enqueued. This is correct for digests: a digest fired at 7:43 AM after a 6:00 AM miss is operationally surprising (operators expect digests on the cadence); the next on-time fire (6 PM) carries the full window via the cutoff-resolution rule below, so no information is lost — only the timing of the message is.
- The routine title prefix `[reporter:digest-trigger]` is the literal string `SKILL.md` step 2 pattern-matches at this commit SHA. Changing this prefix without updating `SKILL.md` step 2 will cause the heartbeat to dispatch to the unknown-trigger-key branch and exit cleanly with no digests emitted.

## Skill input parameters

The trigger reference reads the following inputs from the spawned execution issue's skill-input bag (per Paperclip routine-spawned-issue convention — inputs declared at routine registration are surfaced into each fire's execution issue):

| Input | Default | Notes |
|---|---|---|
| `digestCronExpression` | `0 6,18 * * *` | Documented here for CEO routine-registration override; the registered routine's `cronExpression` is the live source of truth at heartbeat time. This input is informational at the trigger reference level — the cron tick is owned by the routine engine, not by this skill. |
| `digestTimezone` | `America/Denver` | Same role as `digestCronExpression` — documents the registered timezone for CEO override path. The live `routineTriggers.timezone` row is the runtime source of truth. |
| `digestChannelOverride` | `null` | Reserved for future multi-channel work paralleling `slackChannelOverride` in `references/blocker-trigger.md`. v1 of `slack-transport` ([COPAAA-103](/COPAAA/issues/COPAAA-103)) is single-webhook per OQ-5 and ignores per-message channel hints; this input is plumbed through to `slack-transport` invocations but has no effect at v1. Documented here so a future multi-webhook `slack-transport` upgrade does not require a digest-trigger schema bump. |
| `webhookEnvVar` | `SLACK_WEBHOOK_URL` | Passed verbatim to `slack-transport`. Mirrors the `references/blocker-trigger.md` default; both triggers deliver via the same env-var binding by default. |
| `parentIds` | `null` (auto-discover) | Optional array of parent issue identifiers (e.g., `["COPAAA-1", "COPAAA-61"]`) to scope digests to. When `null`, the trigger auto-discovers parents per the resolution rule in step 2 of the algorithm below. When non-null, exactly the listed parents are digested (one Slack message per parent). The auto-discovery path is the operator-default; the explicit list is the smoke / pilot path. |
| `cutoffFallbackHours` | `12` | Used for first-ever run when `heartbeat-runs` returns no prior `finishedAt` for the reporter agent. Twelve hours aligns to the twice-daily cadence — a cold-start digest fired at 6 AM covers the prior 12-hour window, which approximates the operator's mental "since the last digest" expectation. |

Inputs missing from the skill-input bag use the defaults above.

## Composition algorithm

The algorithm is purely API reads up to step 4, then one `slack-transport` call per parent in step 5, then one sentinel-comment write per delivered digest in step 6. Every step has an explicit failure-mode that short-circuits cleanly (no exceptions propagate to `SKILL.md` step 4 — the trigger reference catches them and returns a structured result envelope).

1. **Resolve cutoff timestamp.** Read the reporter agent's most recent finished run from `GET /api/companies/{companyId}/heartbeat-runs?agentId={reporterAgentId}&limit=1`. Filter the response array for entries with `status: "finished"` (NOT `error`, NOT `running`). If at least one finished run exists, set `cutoff = entry.finishedAt`. Otherwise (cold start), set `cutoff = (now - cutoffFallbackHours hours)`. The `now` reference is the heartbeat start time from `Date.now()`. The cutoff bounds the "Accomplished Since Last Digest" window — issues that transitioned to `done` after `cutoff` are counted in section 2 of the digest.

2. **Resolve parent set.**
   - If `parentIds` input is a non-empty array, use it verbatim as the parent set. The trigger does NOT validate that each parent exists — a non-existent parent yields an empty digest at step 3 and the per-parent fire records `{ parent: P, sections: { activeBlockers: [], accomplished: [], remaining: [] }, slackResult: { ok: true, ... } }` in the result envelope. Operators reading the channel see an empty digest and correct the `parentIds` configuration.
   - If `parentIds` is `null` or empty, auto-discover top-level parent issues by issuing `GET /api/companies/{companyId}/issues?limit=500` and filtering client-side for entries where `parentId === null` (top-level issues — no parent issue). This is the **client-side filter** approach (C-3 Code Gate resolution). The `/dashboard` endpoint (BA research-summary §2.5) returns aggregate counts and agent stats, not an `epics[]` array; the source of per-issue parent hierarchy is the issues endpoint. Each client-side-filtered entry's `id` is added to the parent set.
   - If after resolution the parent set is empty (no explicit `parentIds` AND the client-side filter returned no top-level issues), short-circuit immediately with `{ ok: true, alertsSent: 0, cursor: <cutoff>, note: "digest-parent-set-empty" }` — do NOT proceed to the F1 self-check (step 3) with an undefined `firstParent`. The literal note value `digest-parent-set-empty` is the QA2 T-digest-empty-parent pattern-match target (C-2 Code Gate resolution).
   - The parent set is then reduced to a stable order (sort by issue identifier ascending — e.g., `COPAAA-1` before `COPAAA-61`) so per-fire Slack message ordering is deterministic across fires.

3. **Self-check the `?parentId` filter (F1 wiring, B4 anchor).** Issue `GET /api/companies/{companyId}/issues?parentId={firstParent}&limit=1` against the first parent in the resolved set and verify HTTP 200 with an array body. This call is included on every fire to satisfy [COPAAA-106 §F1](/COPAAA/issues/COPAAA-106) Code Gate evidence — *"`?parentId=<COPAAA-61-id>` (or another parent of CTO's choice that returns ≥1 child) showing the digest scoping query returns the expected child set"*. The response body is not consumed by the algorithm; the call exists solely to assert the F1 wiring resolves at the API edge on every fire (so a regression in the `?parentId` parser is caught at the next routine fire rather than at digest-time only). On non-200 response, abort the fire with `{ ok: false, error: "f1_parent_id_self_check_failed", partialAlertsSent: 0 }` — the digest prefers a hard abort here because a broken `?parentId` parser corrupts every subsequent per-parent query in step 4.

4. **Per-parent: assemble three sections.** For each parent `P` in the resolved set (in the order from step 2):
   1. **Section A — Active Blockers.** `GET /api/companies/{companyId}/issues?status=blocked&parentId={P}&limit=200`. The response is the list of currently-blocked children of `P`. For each entry, retain `{ identifier, title, priority, assigneeAgentId }` for composition. The 200-limit page size is the same upper bound as `references/blocker-trigger.md` step 3 — if a single parent has >200 active blockers at digest time the methodology has a deeper problem than the digest format and a single page misses none of the active blockers in any realistic state.
   2. **Section B — Accomplished Since Last Digest.** `GET /api/companies/{companyId}/activity?after={cutoff}&limit=500`. Filter the response array client-side to entries matching all three predicates (mirrors the `references/blocker-trigger.md` predicate shape — same canonical ordering of `action`, `details.status`, `details._previousStatus`):
      - `action === "issue.updated"`.
      - `details.status === "done"`.
      - `details._previousStatus !== "done"`.
      Then cross-reference each filtered entry's `entityId` against parent `P` by reading `details.parentIssueId` if present on the activity payload (BA research-summary §2.3 documents the activity envelope shape) OR falling back to a per-issue lookup (`GET /api/issues/{entityId}` and reading `parentIssueId`) when the activity envelope lacks the parent field. Sort the surviving entries by `createdAt` ascending. Save as section B.
   3. **Section C — Remaining Work.** `GET /api/companies/{companyId}/issues?status=todo,in_progress&parentId={P}&limit=200`. The multi-status `?status=todo,in_progress` exercises the same F1 multi-status parser the blocker-trigger relies on at every fire (cross-trigger F1 reinforcement). The response is the list of children of `P` still in active work. Sort by `priority` (critical → high → medium → low) then by `identifier` ascending for deterministic rendering.

5. **Per-parent: compose and deliver.** For each parent `P` whose three sections were assembled in step 4:
   - Look up `P`'s own record (`GET /api/issues/{P}`) for the digest header — title and priority.
   - Compose the Slack block-kit message body using the `digest-summary` shape from `references/slack-payload-shapes.md`. The shape consumes: parent identifier, parent title, fire wall-clock (formatted in the configured timezone), section A entries, section B entries, section C entries, and a deep link of the form `https://{paperclip-instance-host}/issues/{parentIdentifier}`.
   - Compute the `fireKey`: format `cutoffMoment` as `YYYY-MM-DD-HH` in the configured timezone where `cutoffMoment` is the routine's `nextRunAt` *for this fire* (NOT `now`, NOT `cutoff` from step 1). The `fireKey` deduplicates re-enqueued or replayed fires of the same cron tick across all parents — see step 6 idempotence.
   - Pre-check idempotence: `GET /api/issues/{P}/comments?limit=200` and pattern-match the regex `^\[reporter:digest-summary\] runId=.* fireKey={fireKey} parentId={P} ` against each comment body. If at least one match exists, the digest for this (parent, fireKey) pair was already delivered on a prior fire — skip this parent (do NOT emit a duplicate digest; do NOT re-write the sentinel). The literal phrases `[reporter:digest-summary]` and `fireKey=` MUST appear so QA2's T-digest-idempotence pattern-match works.
   - Invoke `slack-transport` with `{ message: <composed body>, webhookEnvVar: <input.webhookEnvVar> }`. Capture the `slack-transport` result envelope (`{ ok, status, attempts, error?, retryAfter?, note? }` per [COPAAA-103 SKILL.md §5](/COPAAA/issues/COPAAA-103)).

6. **Per-parent: post the idempotence sentinel.** On `slack-transport` `ok: true`, post the sentinel comment on **parent `P`** (NOT on the routine-spawned execution issue): `POST /api/issues/{P}/comments` with body:
   ```
   [reporter:digest-summary] runId={PAPERCLIP_RUN_ID} fireKey={fireKey} parentId={P} status=ok attempts={attempts} sectionCounts=A:{|A|}/B:{|B|}/C:{|C|}
   ```
   The sentinel is written AFTER the Slack delivery succeeds (Slack-first, sentinel-second) — same race-tradeoff as `references/blocker-trigger.md` step 6: at-most-one duplicate digest on a sentinel-write failure, vs the worse alternative of a missed digest if sentinel were written first. The `sectionCounts` suffix surfaces the digest's volume to operators reading the audit trail without re-fetching the Slack message.
   On `ok: false`, do NOT write the sentinel — leave the (parent, fireKey) un-marked so the next fire's pre-check would re-emit. (In practice the next 6 AM / 6 PM fire carries a different `fireKey`, so a transient delivery failure does NOT auto-retry within the cadence; the next fire produces a fresh digest covering an updated window. This is intentional — a "retry stuck digest from 12 hours ago" alert would be operationally noisy.)

7. **Emit the result envelope.** The trigger reference returns to `SKILL.md` step 4 with one of two shapes:
   - Success: `{ ok: true, alertsSent: <integer count of (parent, slack-transport: ok=true) pairs>, cursor: <ISO 8601 timestamp of cutoff used in step 1> }`.
   - Partial / failure: `{ ok: false, error: <literal string describing the failure mode>, partialAlertsSent: <integer count of delivered digests before failure>, cursor: <cutoff used in step 1> }`.
   Both shapes carry `cursor` because `SKILL.md` step 4's fire-summary comment surfaces it for operator inspection. The cursor here is the cutoff (NOT the activity-id cursor used by blocker-trigger), reflecting the digest's window-based semantics.

## Idempotence sentinel format (canonical)

The idempotence sentinel is a comment posted on each **parent issue** (NOT on the routine-spawned execution issue, NOT on individual children) with the literal body shape:

```
[reporter:digest-summary] runId={runId} fireKey={fireKey} parentId={parentId} status=ok attempts={attempts} sectionCounts=A:{|A|}/B:{|B|}/C:{|C|}
```

Where:
- `{runId}` is the value of `PAPERCLIP_RUN_ID` for the heartbeat that delivered the digest.
- `{fireKey}` is the wall-clock fire timestamp formatted as `YYYY-MM-DD-HH` in the configured timezone (e.g., `2026-04-26-06` for the 6 AM Mountain fire on 2026-04-26).
- `{parentId}` is the parent issue's identifier (e.g., `COPAAA-61`).
- `{attempts}` is the `attempts` field from the `slack-transport` result envelope.
- `{|A|}`, `{|B|}`, `{|C|}` are the section sizes (Active Blockers, Accomplished, Remaining).
- `status=ok` is hardcoded — failed deliveries do NOT write a sentinel (see step 6 ordering).

Step 5's pattern match keys on the prefix `[reporter:digest-summary] runId=.* fireKey={fireKey} parentId={parentId} ` (with a trailing space; the `.*` is the regex wildcard for any prior runId — a re-fire on the same parent and fireKey under a different runId is the catch-up edge case that the pre-check correctly suppresses). Operators reading these comments on parent issues see the per-fire audit trail: every twice-daily fire yields one sentinel comment per parent, every parent's sentinel history is the digest cadence record.

## DST correctness (F3 evidence anchor)

The cron-tick computation that schedules `digest-trigger` fires in `America/Denver` lives in `nextCronTickInTimeZone` (`@paperclipai/server/dist/services/routines.js` line 76). The function delegates wall-clock parsing to `Intl.DateTimeFormat` with the IANA timezone string — the same mechanism for both the `assertTimeZone` validation (line 28) and the `getZonedMinuteParts` per-minute zoned breakdown (line 41). The cursor advances in UTC minutes (`floorToMinute` line 36 + `setUTCMinutes(+1)` line 90) and each candidate minute is checked against the IANA-zoned wall-clock parts via `matchesCronMinute` (line 67). Because `Intl.DateTimeFormat` is itself ICU-backed and ICU's IANA tz database is DST-correct, the computation is mathematically sound across spring-forward (`2026-03-08 02:00 → 03:00`, the 02:xx hour does not exist in `America/Denver`) and fall-back (`2026-11-01 01:00 → 02:00 → 01:00 again`, the 01:xx hour occurs twice) transitions. Concretely:
- **Spring-forward** (2026-03-08 in America/Denver): the cursor's UTC-minute walk skips zoned-wall-clock hours that do not exist (e.g., the missing 02:00 hour), so a `0 6 * * *` cron expression naturally fires once per civil day at the post-jump 06:00 wall clock.
- **Fall-back** (2026-11-01 in America/Denver): the cursor's UTC-minute walk visits each unique UTC minute exactly once, so the duplicated 01:00 wall-clock hour fires the cron expression exactly once (corresponding to the first UTC instance, the second UTC instance is the same wall clock but a later UTC, and `matchesCronMinute` is satisfied at the first match).

The Code Gate evidence package binds this prose argument to a verbatim quote of lines 28–93 of `routines.js`, with the function-body inspection serving as F3 proof shape (b) per [COPAAA-106 §F3](/COPAAA/issues/COPAAA-106). No DST-boundary unit test (proof shape (a)) is shipped in this slice — Dev's choice between (a) and (b) is documented in the evidence package; (b) is preferred because the correctness argument applies to ALL cron expressions and ALL IANA timezones uniformly, whereas (a) would only verify the specific `0 6,18 * * *` × `America/Denver` pair.

## Pattern-match phrases (for QA2 Test Gate rubric)

QA2's per-skill Test Gate rubric pattern-matches the literal phrases below in this trigger reference's runtime artifacts:

- `[reporter:digest-summary]` — sentinel-comment prefix (T-digest-sentinel).
- `fireKey=` — sentinel-comment fire-key field (T-digest-firekey).
- `sectionCounts=A:` — sentinel-comment section-counts prefix (T-digest-section-counts).
- `f1_parent_id_self_check_failed` — F1 abort-error literal (T-digest-f1-abort).
- `digest-parent-set-empty` — empty parent set short-circuit literal (T-digest-empty-parent, C-2 Code Gate resolution).
- The composed Slack message phrases (`📊 Project Digest:`, `*Active Blockers*`, `*Accomplished Since Last Digest*`, `*Remaining Work*`) live in `references/slack-payload-shapes.md` `digest-summary` shape — see that file's pattern-match section for the QA2 rubric on the composed body.

Each phrase is a stable test-surface contract. Rephrasing any of them collapses the QA2 test rubric and is gate-failing on the canonical-phrase rule documented in `SKILL.md`'s quality bar.
