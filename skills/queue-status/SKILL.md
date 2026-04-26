---
name: queue-status
description: >
  Heartbeat-time self-introspection skill that posts one structured
  `[queue-status]` comment on the agent's currently-claimed execution issue
  (`PAPERCLIP_TASK_ID`) describing the agent's queue depth and status mix as
  read from `GET /api/agents/me/inbox-lite` (BA research summary §2.5). The
  comment line is the contract surface the
  [`copperforge-reporter` quiet-stall trigger (B5)](/COPAAA/issues/COPAAA-108)
  pattern-matches against — see [`references/output-schema.md`](references/output-schema.md)
  for the verbatim line format, regex, and dedupe key. Use this skill when
  invoked by an agent's end-of-heartbeat hook (the hook itself ships in
  [B6 — COPAAA-107](/COPAAA/issues/COPAAA-107), not here) to emit one
  parseable queue-status comment per heartbeat run; this skill performs zero
  reporter-side correlation, zero stall detection, and zero Slack delivery —
  those concerns live in `copperforge-reporter`.
---

# queue-status

Use this skill when an agent's end-of-heartbeat hook needs to emit one structured `[queue-status]` comment summarising its own queue depth and status mix on the issue the heartbeat is currently running against. The skill is a pure self-introspection primitive: it has no awareness of CopperForge methodology, of which agents matter to the reporter, of stall thresholds, or of Slack delivery. The contract surface the [`copperforge-reporter` quiet-stall trigger (B5 — COPAAA-108)](/COPAAA/issues/COPAAA-108) pattern-matches lives in [`references/output-schema.md`](references/output-schema.md); this body owns only the read-classify-post heartbeat workflow. Per the [COPAAA-61](/COPAAA/issues/COPAAA-61) parent issue board direction, the reporter is the only place CopperForge methodology awareness lives in the delivery pipeline — `queue-status` carries no methodology and any agent in any company can invoke it. Per the [COPAAA-105](/COPAAA/issues/COPAAA-105) issue scope, the per-role invocation hook (the end-of-heartbeat edit that adds a queue-status invoke step to each role's `HEARTBEAT.md`) is **B6** ([COPAAA-107](/COPAAA/issues/COPAAA-107)), not this skill — this skill ships the body alone and is registered into the company at this commit SHA so B6 can wire it.

## Preconditions

- The skill is invoked from inside an agent's heartbeat run. The Paperclip-injected environment variables `PAPERCLIP_TASK_ID`, `PAPERCLIP_RUN_ID`, and `PAPERCLIP_AGENT_ID` are populated; if any of the three is missing or empty, the skill exits cleanly with a no-op log (workflow step 1) — heartbeat continues normally and is not blocked by an inability to self-report.
- The skill makes exactly one read call (`GET /api/agents/me/inbox-lite` per BA research summary §2.5) and exactly one write call (`POST /api/issues/{PAPERCLIP_TASK_ID}/comments` per BA §2.3) per invocation. It does NOT call any other Paperclip endpoint, does NOT invoke `slack-transport`, does NOT touch disk beyond reading its own SKILL.md context, and does NOT mutate any issue field other than appending one comment.
- Outbound HTTPS uses the Paperclip auth pair `Authorization: Bearer $PAPERCLIP_API_KEY` (read + write) plus `X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID` (write only) per BA §2.6. The skill does NOT use any other credential, does NOT call `/api/agents/:id/runtime-state` (Board-gated per BA §2.4), and does NOT call `/api/companies/:companyId/secrets`.
- The contract `[queue-status] depth=N ...` line format is byte-stable per [`references/output-schema.md`](references/output-schema.md). Reporter B5 ([COPAAA-108](/COPAAA/issues/COPAAA-108)) pattern-matches against the regex documented there; any change to the line format is a contract change that requires a coordinated B3 + B5 update.

## Workflow

1. **Read heartbeat context.** Read `PAPERCLIP_TASK_ID`, `PAPERCLIP_RUN_ID`, and `PAPERCLIP_AGENT_ID` from the environment. If any is missing or empty after `.trim()`, log the literal phrase `[queue-status] no-op: missing heartbeat env (taskId|runId|agentId)` (substituting the missing variable name) and exit cleanly. The heartbeat runs without a `PAPERCLIP_TASK_ID` (e.g., autonomous heartbeats with no claimed issue) have no comment target — the skill does NOT fall back to any other issue, does NOT search for a "default" target, and does NOT throw. The literal phrase `[queue-status] no-op:` MUST appear so QA2's T1 can pattern-match it.

2. **Fetch inbox-lite.** Issue `GET {PAPERCLIP_API_URL}/api/agents/me/inbox-lite` with `Authorization: Bearer $PAPERCLIP_API_KEY`. Timeout 10 seconds. The response is a JSON array of compact issue records each containing at least `{ id, identifier, title, status, priority, projectId, parentId, updatedAt }` (BA §2.5). On any non-200 response or network error, log the literal phrase `[queue-status] no-op: inbox-lite read failed status={status}` (with `{status}` set to the HTTP code or `null` for network errors) and exit cleanly. The skill does NOT retry — heartbeat-time self-reports are sampled per-fire, and a single missed sample is recoverable on the next heartbeat. The literal phrase `inbox-lite read failed` MUST appear so QA2's T2 can pattern-match it.

3. **Compute depth and status mix.** `depth` is the integer length of the inbox-lite array. The `mix` field enumerates counts for the canonical Paperclip status set in this exact order: `todo`, `in_progress`, `in_review`, `blocked`, `done`, `cancelled`, `other`. Any inbox-lite record whose `status` value is not one of the first six canonical names is counted in `other` (forward-compatible with future status additions per BA §2.5; emitting unknown statuses verbatim would break the reporter's stable-key regex). All seven counts are always emitted including zeros — the comma-separated key:value list is deterministic-order so reporter regex doesn't need a key-set permutation table. If `depth === 0`, all six canonical counts are zero and `other` is zero; the comment is still posted (a silent agent is also a signal the reporter wants to see).

4. **Compose the line.** Format one line per [`references/output-schema.md`](references/output-schema.md):
   ```
   [queue-status] depth=<N> mix=todo:<a>,in_progress:<b>,in_review:<c>,blocked:<d>,done:<e>,cancelled:<f>,other:<g> agentId=<PAPERCLIP_AGENT_ID> runId=<PAPERCLIP_RUN_ID> ts=<ISO8601-UTC>
   ```
   - `<N>` is the integer from step 3.
   - `<a>..<g>` are the integer counts from step 3 in the exact order `todo,in_progress,in_review,blocked,done,cancelled,other`.
   - `<PAPERCLIP_AGENT_ID>` and `<PAPERCLIP_RUN_ID>` are the verbatim env-var values from step 1 (UUIDs).
   - `<ISO8601-UTC>` is the current UTC time formatted as `YYYY-MM-DDTHH:MM:SS.sssZ` (e.g., `2026-04-26T17:30:00.000Z`). Use `new Date().toISOString()` or platform equivalent — no `Intl.DateTimeFormat` timezone formatting is needed (this timestamp is wire-format, not display).
   The line is one line, no leading or trailing whitespace, no markdown formatting around it (the body of the comment IS the line). Reporter B5's regex anchors on `^\[queue-status\] ` so the line MUST be the first character of the comment body.

5. **Post the comment.** Issue `POST {PAPERCLIP_API_URL}/api/issues/{PAPERCLIP_TASK_ID}/comments` with `Authorization: Bearer $PAPERCLIP_API_KEY`, `X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID`, `Content-Type: application/json`, and body `{ "text": "<line from step 4>" }`. Timeout 10 seconds. On any non-2xx response or network error, log the literal phrase `[queue-status] no-op: comment post failed status={status}` (with `{status}` set to the HTTP code or `null` for network errors) and exit cleanly — the skill does NOT retry, does NOT mutate `PAPERCLIP_TASK_ID` status, and does NOT escalate to the operator. A failed self-report is one missed sample; the next heartbeat re-attempts. The literal phrase `comment post failed` MUST appear so QA2's T6 can pattern-match it.

6. **Exit.** On success, log the literal phrase `[queue-status] posted depth={N} runId={runId} taskId={taskId}` (with the substituted values) and exit cleanly. The heartbeat continues — `queue-status` does NOT mark `PAPERCLIP_TASK_ID` as `done`, does NOT advance any state machine, and does NOT comment on any other issue. The skill's only side effect is the one comment posted in step 5.

## Quality bar

- Exactly one `GET /api/agents/me/inbox-lite` and exactly one `POST /api/issues/{PAPERCLIP_TASK_ID}/comments` per invocation. No retry on either, no second read, no second write. A test or evidence package that observes a second read or a second write per invocation is gate-failing on the one-read-one-write rule.
- Zero calls to `/api/agents/:id/runtime-state` (Board-gated per BA §2.4 — would 403 from a non-Board agent and is unnecessary for self-introspection because `me/inbox-lite` is the calling agent's own queue). Adding that call is gate-failing on the unauthorized-endpoint rule.
- Zero calls to `slack-transport`. This skill is purely a Paperclip-comment producer; Slack delivery happens downstream in `copperforge-reporter` ([COPAAA-104](/COPAAA/issues/COPAAA-104) / [COPAAA-108](/COPAAA/issues/COPAAA-108)) when the reporter pattern-matches `[queue-status]` lines and decides a quiet-stall alert is warranted. A code path that POSTs directly to Slack from this skill is gate-failing on T-direct-post.
- The output line is byte-stable per [`references/output-schema.md`](references/output-schema.md). The literal prefix `[queue-status] ` MUST be the first 16 characters of the comment body. The keys `depth=`, `mix=`, `agentId=`, `runId=`, `ts=` MUST appear in that exact order separated by single ASCII spaces. The mix-list keys (`todo,in_progress,in_review,blocked,done,cancelled,other`) MUST appear in that exact order separated by single commas with no spaces. Reordering any key — including alphabetising the mix list — is gate-failing because reporter B5's regex is positional in v1.
- All seven `mix` counts are always emitted, including zero counts. Suppressing zero-valued statuses ("compact form") is gate-failing because the reporter's regex assumes a fixed seven-tuple and would mis-parse a six-key form as a different schema version.
- Unknown statuses (anything outside the six canonical names) are aggregated into the `other` bucket, NOT emitted as new keys. Adding a new key for a future status (e.g., `mix=...,review_needed:1,other:0`) is gate-failing on the stable-key rule. Future status additions land via a coordinated B3 + B5 schema bump under a separate ticket, not via silent extension.
- Every reject / no-op path uses one of the literal phrases enumerated above (`[queue-status] no-op: missing heartbeat env`, `[queue-status] no-op: inbox-lite read failed status=`, `[queue-status] no-op: comment post failed status=`, `[queue-status] posted depth=`). QA2 pattern-matches these in T1, T2, T6, T7; rephrasing collapses the test surface.
- `PAPERCLIP_API_KEY` and `PAPERCLIP_RUN_ID` are touched only by the two HTTP calls in steps 2 and 5. They are NOT logged, NOT included in any return field, NOT echoed to stdout/stderr beyond the standard Paperclip skill log envelope, and NOT passed into any other call. The skill does NOT log the inbox-lite response body — only the count and the derived mix tuple are surfaced.
- Per-skill README row update is mandatory at Code Gate per the [per_skill_readme_row_rule.md](https://github.com/jacobfogg/CopperForge-Method/blob/main/methodology/) rule (COPAAA-19 B4 ratified). The `queue-status` row in repo `README.md` lands in the same coding commit as this SKILL.md and is reaffirmed in the Code Gate evidence package.
- Frontmatter contains `name` and `description` only — no `model`, no `allowed-tools`, no input schema. This mirrors the four canonical Paperclip alpha skills and the sibling `slack-transport` / `copperforge-reporter` skills (BA research summary §4: *"all four use only `name` and `description`"*).
- The skill body does not perform any reporter-side correlation, dedupe, or stall detection. The reporter B5 trigger ([COPAAA-108](/COPAAA/issues/COPAAA-108)) consumes a stream of `[queue-status]` comments across heartbeats and decides whether a stall warrants a Slack alert; B3 emits the raw signal only. Any "is the agent stalled?" logic added to this body is gate-failing on the body-narrowness rule (mirrors the COPAAA-61 board direction "Resist the temptation to merge" between layers).

## References

- [`references/output-schema.md`](references/output-schema.md) — verbatim line format, the parsing regex reporter B5 uses, the canonical seven-tuple `mix` ordering, and the `(agentId, runId)` dedupe key. This file is the **contract** between B3 (this skill, producer) and B5 ([COPAAA-108](/COPAAA/issues/COPAAA-108), consumer); changes to it require a coordinated update of both skills under a separate ticket.

## v1 scope summary (what does and does not ship at this commit SHA)

This skill body and the one `references/` file referenced above are the entirety of B3's v1 ship. Specifically NOT shipped at v1 (deferred per [COPAAA-85 plan §B6 / §B5](/COPAAA/issues/COPAAA-85#document-plan) and the [COPAAA-105 issue scope](/COPAAA/issues/COPAAA-105)):

- The end-of-heartbeat invocation hook (the per-role `HEARTBEAT.md` edit that adds a queue-status invoke step) is **B6** ([COPAAA-107](/COPAAA/issues/COPAAA-107)). B6 wires this skill into each role's heartbeat without touching the SKILL.md body.
- The reporter-side quiet-stall trigger (the consumer that pattern-matches `[queue-status]` lines and decides whether to alert) is **B5** ([COPAAA-108](/COPAAA/issues/COPAAA-108)). B5 adds `references/quiet-stall-trigger.md` to the `copperforge-reporter` skill and registers a new routine; B5 is downstream of B3 *done* AND B6 *done* per the parent issue's vertical-slice ordering ("`queue-status` data must accumulate ≥4h before quiet-stall has data to operate on").
- Per-parent-issue breakdowns, per-priority breakdowns, and `activeRun` correlation are NOT in v1's `mix` line. Adding fields to the contract is a coordinated B3 + B5 schema bump and is out of scope for this ticket.
