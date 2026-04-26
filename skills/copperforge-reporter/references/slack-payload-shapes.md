# copperforge-reporter — Slack payload shapes (block-kit)

This reference owns every block-kit composition the reporter emits. `SKILL.md` and the per-trigger references (e.g., `blocker-trigger.md`) consume these shapes by name and pass through the result to `slack-transport` as the `message` argument. v1 ships exactly one shape — `blocker-alert`. B4 will add `digest-summary` (three-section daily digest) and B5 will add `quiet-stall-alert` to this same file without touching `SKILL.md` or `blocker-trigger.md`.

This file is the single place the reporter knows about Slack block-kit. `slack-transport` is intentionally Slack-shape-agnostic ([COPAAA-103 SKILL.md preconditions](/COPAAA/issues/COPAAA-103): *"the skill does NOT validate the block-kit shape"*) — composition lives upstream, here. Any block-kit logic appearing in `SKILL.md` or in `blocker-trigger.md` is gate-failing on the body-narrowness rule documented in `SKILL.md`'s quality bar.

## Block-kit reference (cited authority)

- Slack incoming-webhook payload contract: `{ text?, blocks?, attachments? }` where `blocks` is the rich shape and `text` is the notification fallback. Source: <https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks> (BA research-summary §1.1, Verified).
- Maximum **50 blocks per message**. Source: <https://docs.slack.dev/reference/block-kit/blocks> (BA research-summary §1.3, Verified).
- Maximum **100 attachments per message** — the reporter does NOT use attachments at v1 (block-kit `blocks` carries the entire body). Source: <https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks> (BA research-summary §1.3, Verified).
- Per-block character limits (`mrkdwn` section text commonly cited at 3000 chars; header `plain_text` at 150 chars; field value at 2000 chars) are documented per block-type but were NOT directly verified by BA in §1.3 (Assumption — BA flagged this in OQ-2). Dev re-verifies these limits at this skill's Code Gate per [COPAAA-104 Code Gate evidence](/COPAAA/issues/COPAAA-104) and updates the truncation budget below if Slack documentation shows a different number. Until that re-verification lands, the truncation budget below uses the conservative cited values.

## Truncation budget (v1, conservative; subject to Code Gate re-verification)

The reporter applies these per-field caps before composition:

| Field | Cap | Source |
|---|---|---|
| Header `plain_text` (block-kit `header` block) | 150 chars | Per-block-type page (Assumption — BA OQ-2; re-verified at Code Gate) |
| Section `mrkdwn` text | 3000 chars | Per-block-type page (Assumption — BA OQ-2; re-verified at Code Gate) |
| Field `mrkdwn` value (in section `fields[]`) | 2000 chars | Per-block-type page (Assumption — BA OQ-2; re-verified at Code Gate) |
| Issue title (rendered into header) | 100 chars | Reporter-internal cap; well under the header 150-char limit, leaves budget for the prefix (e.g., `[BLOCKED] COPAAA-104: …`) |
| Blocker-context excerpt (rendered into a section block) | 2500 chars | Reporter-internal cap; well under the 3000-char section cap, leaves budget for surrounding labels |

Truncation rule when a field exceeds its cap: cut to `cap - 1` characters, append the literal character `…` (U+2026 horizontal ellipsis). Truncation is applied AFTER mrkdwn escape, so escape sequences are not split. The literal `…` is the truncation marker QA2 pattern-matches for T-truncation.

## mrkdwn escape rules

Slack's `mrkdwn` formatting interprets `*`, `_`, `~`, `` ` ``, and `<...>` as control characters. Issue titles and comment bodies pulled from Paperclip routinely contain these characters (e.g., `_status_` markers in blocker comments). The reporter applies the following escape map before inlining ANY Paperclip-sourced string into a `mrkdwn` block:

| Source character | Escaped form |
|---|---|
| `&` | `&amp;` |
| `<` | `&lt;` |
| `>` | `&gt;` |

(Slack's documented mrkdwn escape rules cover only these three — the asterisk / underscore / backtick characters are NOT escapable inline; they MUST be wrapped in inline code spans `\`...\`` if literal rendering is required. The reporter's v1 strategy is the simpler one: do NOT attempt to render literal `*`/`_`/`` ` `` from Paperclip strings — accept that some comment bodies render with stray bold/italic. This is acceptable for v1 because the alert's job is to surface the blocker, not to faithfully reproduce comment formatting.)

URLs inlined into a `mrkdwn` field follow the link shape `<{url}|{label}>`. The label is escaped per the table above; the URL is NOT escaped (Slack expects a literal URL).

## Shape: `blocker-alert` (v1)

Consumed by `references/blocker-trigger.md` step 6 ("Compose and deliver"). Inputs:

- `issueIdentifier` (string) — e.g., `COPAAA-104`.
- `issueTitle` (string) — full issue title.
- `issueStatus` (string) — always `blocked` for this shape (the trigger only fires on issues that have just transitioned into `blocked`).
- `issuePriority` (string) — `critical | high | medium | low`.
- `assigneeAgentName` (string | null) — the agent name (NOT the agent id) of the issue's assignee at fire time. Null if unassigned.
- `blockerContext` (string) — the latest comment body on the blocked issue, OR the issue's `description` field if no comments exist. Truncated per the table above before composition.
- `deepLink` (string) — `https://{paperclip-instance-host}/issues/{issueIdentifier}`. The instance host is read from `PAPERCLIP_API_URL` (env var) and rewritten from the API host to the UI host per the Paperclip URL convention; if the rewrite cannot be performed safely, the deep link uses the API URL as a fallback (operators can still reach the issue via the API path).

Composed body (block-kit object passed verbatim to `slack-transport` as `message`):

```json
{
  "text": "[BLOCKED] {issueIdentifier}: {issueTitle}",
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "🚧 [BLOCKED] {issueIdentifier}: {issueTitle}",
        "emoji": true
      }
    },
    {
      "type": "section",
      "fields": [
        {
          "type": "mrkdwn",
          "text": "*Status*\nblocked"
        },
        {
          "type": "mrkdwn",
          "text": "*Priority*\n{issuePriority}"
        },
        {
          "type": "mrkdwn",
          "text": "*Assignee*\n{assigneeAgentName | 'unassigned'}"
        },
        {
          "type": "mrkdwn",
          "text": "*Issue*\n<{deepLink}|{issueIdentifier}>"
        }
      ]
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*What's blocking*\n{blockerContext}"
      }
    },
    {
      "type": "context",
      "elements": [
        {
          "type": "mrkdwn",
          "text": "Reporter blocker-trigger fire — runId `{runId}` — activityId `{activityId}`"
        }
      ]
    }
  ]
}
```

**Composition rules:**

- The `text` field at top-level is the notification fallback per Slack's webhook contract; clients that do not render block-kit (mobile push notifications, screen readers) display this string. It is intentionally short — `[BLOCKED] {issueIdentifier}: {issueTitle}` — and capped at 200 chars total. Truncate the `issueTitle` portion if the combined length exceeds 200 chars.
- The header block uses `🚧` (U+1F6A7 construction sign) as a visual marker. Slack renders emoji shortcodes (`:construction:`) AND raw unicode emojis equivalently in `plain_text` blocks when `emoji: true`; the raw unicode form is preferred for stability across Slack's emoji-set updates.
- The `fields[]` array in the second `section` MUST contain exactly four entries (status, priority, assignee, issue). Adding more fields here without updating `blocker-trigger.md`'s composition step is gate-failing on the shape-stability rule.
- The third block (`section` with the `*What's blocking*` heading) carries the truncated `blockerContext`. If the comment body is empty (e.g., issue went `blocked` without a blocker comment), the value is the literal string `_(no blocker context comment found — see issue description)_` (italicized via mrkdwn). The italicization signals operator-readable absence-of-data without leaving the field empty, which Slack rejects on some block-kit shapes.
- The `context` block carries the run-id and activity-id metadata. This is intentionally a `context` block (not a `section`) because Slack renders `context` in a smaller font and dimmer color — operationally the right cue that this is metadata, not the alert content. Operators reading the audit trail in the channel can correlate the fire to the per-issue idempotence sentinel via `activityId`.
- Total block count: 4 (header + section-fields + section-context + context-metadata). Well under the 50-block-per-message Slack cap (BA §1.3, Verified).

## Shape stability and additivity

Future trigger references (B4 `digest-summary`, B5 `quiet-stall-alert`) add NEW shape definitions in this file as new H2 sections (e.g., `## Shape: digest-summary (v2)`). They do NOT modify the `blocker-alert` shape. This additivity is the same principle `SKILL.md` step 2 follows for the dispatch table — once a shape ships at a given commit SHA, its block-kit body is contractually frozen until a documented schema bump under the corresponding per-skill Code Gate (parallel to the [COPAAA-103](/COPAAA/issues/COPAAA-103) result-envelope shape-stability discipline).

The shape-stability invariant exists because operators tune their channel-monitoring rules (e.g., Slack channel routing, escalation triggers, on-call paging) against specific block-kit field positions and labels. Silent reshaping of the v1 `blocker-alert` body would invalidate those rules without warning; explicit schema bumps give operators a heads-up.

## Pattern-match phrases (for QA2 Test Gate rubric)

QA2's per-skill Test Gate rubric pattern-matches the literal phrases below in the `blocker-alert` shape's composed output:

- `🚧 [BLOCKED]` — header prefix (T-shape-header).
- `*What's blocking*` — third section heading (T-shape-context-heading).
- `Reporter blocker-trigger fire — runId` — context-block prefix (T-shape-metadata).
- `…` (U+2026 ellipsis) — truncation marker, present iff a per-field cap was hit (T-shape-truncation).
- `_(no blocker context comment found — see issue description)_` — empty-context fallback (T-shape-empty-context).

Each phrase is a stable test-surface contract. Rephrasing any of them collapses the QA2 test rubric and is gate-failing on the canonical-phrase rule documented in `SKILL.md`'s quality bar.
