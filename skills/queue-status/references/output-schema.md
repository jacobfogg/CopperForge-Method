# queue-status output schema

This file documents the verbatim line format, the parsing regex, and the dedupe key for the `[queue-status]` heartbeat-time comment emitted by [`queue-status` SKILL.md](../SKILL.md). It is the contract between **B3** ([COPAAA-105](/COPAAA/issues/COPAAA-105), the producer) and **B5** ([COPAAA-108](/COPAAA/issues/COPAAA-108), the `copperforge-reporter` quiet-stall trigger, the consumer). Any change to this file is a contract change and requires a coordinated B3 + B5 update under a separate ticket.

## Line format

Each invocation of `queue-status` posts exactly one comment whose `text` field is **one line** with the following byte-stable structure:

```
[queue-status] depth=<N> mix=todo:<a>,in_progress:<b>,in_review:<c>,blocked:<d>,done:<e>,cancelled:<f>,other:<g> agentId=<UUID> runId=<UUID> ts=<ISO8601-UTC>
```

Where:

| Token | Type | Description |
|---|---|---|
| `[queue-status] ` | literal prefix (16 chars incl. trailing space) | Stable anchor for reporter regex; MUST be the first 16 characters of the comment body. |
| `depth=<N>` | integer | Total count of issues returned by `GET /api/agents/me/inbox-lite`. |
| `mix=...` | comma-separated key:value list | The seven canonical status counts in the exact fixed order below. All seven are always emitted including zeros. |
| `agentId=<UUID>` | UUID v4 string | Verbatim `PAPERCLIP_AGENT_ID` env var (the calling agent). |
| `runId=<UUID>` | UUID v4 string | Verbatim `PAPERCLIP_RUN_ID` env var (the heartbeat run that emitted this line). |
| `ts=<ISO8601-UTC>` | ISO 8601 UTC timestamp | Format `YYYY-MM-DDTHH:MM:SS.sssZ` (output of `new Date().toISOString()`). |

**Token separation:** single ASCII space (`U+0020`) between the five top-level tokens (`[queue-status] `, `depth=...`, `mix=...`, `agentId=...`, `runId=...`, `ts=...`). Single ASCII comma (no surrounding spaces) between mix entries. Single ASCII colon between mix key and value.

**Whitespace:** no leading whitespace, no trailing whitespace, no internal newlines. The line MUST be representable as one log row. The Paperclip comments API stores `text` verbatim including trailing newlines, so the producer trims any trailing newline before posting.

**Markdown:** none. The body of the comment is the raw line. No surrounding code fence, no bold/italic markers, no link syntax. Reporter B5 reads `comment.text` directly from the API response and pattern-matches with the regex below.

## Mix ordering (canonical seven-tuple)

The seven mix keys appear in exactly this order, separated by single commas:

1. `todo`
2. `in_progress`
3. `in_review`
4. `blocked`
5. `done`
6. `cancelled`
7. `other`

The first six names match the canonical Paperclip status enum (`todo`, `in_progress`, `in_review`, `blocked`, `done`, `cancelled`). The seventh bucket (`other`) aggregates any inbox-lite record whose `status` value is not one of those six. The `other` bucket is the forward-compatibility hatch: when Paperclip introduces a new status, the existing seven-tuple regex still parses (the new status's count rolls into `other`) until a coordinated B3 + B5 schema bump introduces the new key in a new schema version.

**No alphabetisation.** The order is positional, not alphabetic. Reporter B5's regex is positional in v1.

**No suppression of zeros.** A `depth=0` line still emits all seven counts as `0`, e.g.:

```
[queue-status] depth=0 mix=todo:0,in_progress:0,in_review:0,blocked:0,done:0,cancelled:0,other:0 agentId=6f4fb85e-6e4b-4e9b-a131-f706839d42d1 runId=7c12ae84-ec87-4c44-8a3c-4f4ed6c8048f ts=2026-04-26T17:30:00.000Z
```

## Parsing regex

Reporter B5 parses each `[queue-status]` comment with the following PCRE-compatible regex. The regex is anchored on the literal prefix and uses named capture groups for the depth, the seven mix counts, the agentId, the runId, and the timestamp:

```
^\[queue-status\] depth=(?P<depth>\d+) mix=todo:(?P<todo>\d+),in_progress:(?P<in_progress>\d+),in_review:(?P<in_review>\d+),blocked:(?P<blocked>\d+),done:(?P<done>\d+),cancelled:(?P<cancelled>\d+),other:(?P<other>\d+) agentId=(?P<agentId>[0-9a-f-]{36}) runId=(?P<runId>[0-9a-f-]{36}) ts=(?P<ts>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z)$
```

**Anchors.** `^` matches the start of the comment body (NOT the start of a line within a larger body — the line is the entire body). `$` matches the end of the comment body. A comment whose body has trailing whitespace, a trailing newline, or any prose after the `ts=...Z` token is **not** matched and is ignored by the reporter; the producer SKILL.md guarantees no such trailing content.

**UUID class.** `[0-9a-f-]{36}` matches the canonical lowercase UUID-v4 form (32 hex chars + 4 hyphens). The producer emits agent and run UUIDs verbatim from the Paperclip-injected env vars, which are lowercase canonical form.

**Mix-count widths.** `\d+` allows any number of digits per count. The producer does NOT zero-pad; counts are emitted as the natural integer.

**Schema version.** v1 carries no explicit `schema=` token. Reporter B5 treats any line that matches the v1 regex as schema v1. A future v2 schema bump (e.g., adding a per-priority breakdown) will add `schema=v2` after the `[queue-status] ` prefix and Reporter B5 will branch on the schema token. Producers MUST NOT emit a `schema=` token in v1.

## Dedupe key

The reporter B5 quiet-stall trigger ingests `[queue-status]` comments across heartbeats and dedupes on the **two-tuple `(agentId, runId)`**. Each heartbeat run posts at most one queue-status comment per invocation (per the producer's one-write-per-invocation quality bar), but the consumer may re-read the same issue's comment list multiple times across reporter fires; deduping on `(agentId, runId)` ensures the same heartbeat-emitted line is counted once across reporter fires.

`(agentId, ts)` is **not** the dedupe key — two queue-status invocations from the same agent in the same heartbeat run (which the producer rules out, but which a future bug could produce) would have identical `runId` and therefore dedupe correctly; whereas two invocations within the same millisecond would share `ts` and incorrectly collapse.

`commentId` (the Paperclip comment record id) is **not** the dedupe key either — comment ids are unique per write but two reporter fires reading the same issue see the same commentId, so deduping on commentId is a more aggressive constraint than `(agentId, runId)`. The contract documents `(agentId, runId)` so a future producer that posts via a different mechanism (e.g., a routine-spawned agent) still dedupes correctly.

## Comment target

The producer posts the comment via `POST /api/issues/{PAPERCLIP_TASK_ID}/comments`, where `PAPERCLIP_TASK_ID` is the execution issue id of the heartbeat run that invoked `queue-status`. **There is exactly one queue-status comment per heartbeat run, on the issue the heartbeat is running against** — not on every issue in the agent's inbox, not on the parent epic, not on any reporter-owned issue. Reporter B5's scan strategy reads queue-status comments from the union of issues the reporter agent's company knows about (typically by walking the inbox-lite of every active agent across recent heartbeat-runs); the SKILL.md body of B3 does NOT pre-aggregate or cross-issue-fan-out.

## Worked example

An agent with `agentId=6f4fb85e-6e4b-4e9b-a131-f706839d42d1` runs a heartbeat (`runId=7c12ae84-ec87-4c44-8a3c-4f4ed6c8048f`) at `2026-04-26T17:30:00.000Z` against execution issue `COPAAA-105`. Its inbox-lite returns 7 issues: 3 `blocked`, 2 `todo`, 1 `in_progress`, 1 `done`. The producer posts the following comment to `COPAAA-105`:

```
[queue-status] depth=7 mix=todo:2,in_progress:1,in_review:0,blocked:3,done:1,cancelled:0,other:0 agentId=6f4fb85e-6e4b-4e9b-a131-f706839d42d1 runId=7c12ae84-ec87-4c44-8a3c-4f4ed6c8048f ts=2026-04-26T17:30:00.000Z
```

The parsing regex matches with named captures: `depth=7`, `todo=2`, `in_progress=1`, `in_review=0`, `blocked=3`, `done=1`, `cancelled=0`, `other=0`, `agentId=6f4fb85e-…`, `runId=7c12ae84-…`, `ts=2026-04-26T17:30:00.000Z`. Reporter B5 dedupes on `(agentId, runId) = (6f4fb85e-…, 7c12ae84-…)`.

## Out of scope for v1

These are deliberately not part of the contract at v1 and any producer that emits them is gate-failing on the contract-stability rule:

- `schema=` token (reserved for future bumps; v1 has no explicit schema token).
- Per-priority breakdown (e.g., `pri=high:2,medium:5,low:0`).
- Per-parent breakdown (e.g., `parents=COPAAA-61:3,COPAAA-1:4`).
- `activeRun` correlation (whether the agent is currently executing a run).
- Companion structured fields (e.g., a JSON appendix after the line).
- Multi-line comment bodies. The line is the entire body.

Future expansion lands via a coordinated B3 + B5 schema bump under a separate ticket; v1 ships the seven-tuple-only contract above.
