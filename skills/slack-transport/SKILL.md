---
name: slack-transport
description: >
  Deliver an already-composed Slack message to a single Slack incoming-webhook
  URL via one outbound HTTPS POST per invocation, with the four-class retry
  policy from the COPAAA-62 BA research summary §1.5 (HTTP 200 ok / 4xx
  named-error / 429 with Retry-After / 5xx or network error with exponential
  backoff), capped at 3 retries. Read the webhook URL from a caller-named
  environment variable populated at heartbeat-launch by a Paperclip
  `secret_ref` binding (env-var-injection path per research summary §5.1),
  and return a structured `{ok, attempts, ...}` result the caller acts on.
  Use this skill when a caller needs to ship a Slack-shaped message body to
  one webhook destination — block-kit composition, channel routing,
  message templating, and methodology-aware decisions of "what to send and
  when" all live upstream in `copperforge-reporter` or any other caller;
  this skill performs zero composition and zero CopperForge logic.
---

# slack-transport

Use this skill when a caller has an already-composed Slack message body and needs to deliver it to a single Slack incoming-webhook URL. The skill is a pure delivery primitive: it has no awareness of CopperForge, the methodology, what is being delivered, or where the webhook points. The four-class retry policy lifted from the COPAAA-62 BA research summary §1.5 (200 → ok; 4xx → permanent named-error; 429 → honor `Retry-After`; 5xx or network → exponential backoff) is implemented inline below — it is the entire decision surface this skill owns. Composition shapes (block-kit `blocks`, `attachments`, `text`, message templates), channel routing across multiple webhooks, and methodology-aware decisions of "what to send and when" all live upstream in [`copperforge-reporter`](/COPAAA/issues/COPAAA-85#document-plan) or any other caller, per the [COPAAA-61](/COPAAA/issues/COPAAA-61) parent issue board direction ("Resist the temptation to merge slack-transport and copperforge-reporter"). This skill is intentionally narrow so it can be reused for any future delivery need.

## Preconditions

- The caller passes a `message` argument that is a JSON-serializable object containing the Slack message body to deliver (e.g., `{ text, blocks, attachments }` per Slack's incoming-webhook contract). The skill does NOT validate the block-kit shape — Slack returns a 4xx named-error such as `invalid_blocks` if the body is malformed and that surfaces verbatim through the result. If `message` is missing, null, or not an object, the skill rejects with the literal phrase `message argument is required` (workflow step 1).
- The caller passes a `webhookEnvVar` argument naming the environment variable the skill reads at run time to resolve the webhook URL. The default is `SLACK_WEBHOOK_URL`. The caller MUST configure a Paperclip company-level `secret_ref` binding such that the named environment variable is populated with the resolved Slack incoming-webhook URL at heartbeat-launch (env-var-injection mechanism per BA research summary §5.1). The exact production binding name (`SLACK_WEBHOOK_URL` for v1 single-webhook, or a custom name for future multi-channel builds) is locked in by the [COPAAA-103 OQ-5 single-webhook ack](/COPAAA/issues/COPAAA-103) from CEO; the skill body is parametric on the env-var name so the binding-name decision does not block coding.
- The named environment variable resolves to a non-empty string at run time. If unset or empty the skill rejects with the literal phrase `webhook env var {name} not set` (workflow step 2). The skill does NOT log, echo, or pass the resolved URL into any other field of its output — the URL is sensitive and is touched only by the outbound POST.
- This skill makes exactly one outbound HTTPS POST to the resolved webhook URL per attempt (1 + up to 3 retries = at most 4 wire calls per invocation). It does NOT call the Paperclip API, does NOT touch disk, does NOT read any Paperclip company/agent record, and does NOT compose Slack message content. Any test or evidence package that observes a Paperclip API call, a disk write, or a second webhook URL being touched in the same invocation is gate-failing on the quality-bar bullets below.
- Outbound HTTPS uses anonymous transport — no `Authorization` header, no auth credential beyond Slack's webhook token embedded in the URL path (which Slack rotates per webhook). Adding an `Authorization` header on the webhook POST is gate-failing.

## Workflow

1. **Argument validation.** Read `message` from invocation args. If `message` is missing, null, or fails a basic "is a JSON-serializable object" check (i.e., `JSON.stringify(message)` throws or yields `"null"` / `"undefined"`), reject with the literal message `"slack-transport: message argument is required"` and exit before any environment read or network call. The literal phrase `message argument is required` MUST appear so QA2's T1 can pattern-match it. Do not attempt to "auto-wrap" a string `message` into `{text}` — wrapping is composition and composition is upstream.

2. **Resolve webhook URL.** Read `webhookEnvVar` from invocation args; default to the literal string `"SLACK_WEBHOOK_URL"` if missing or empty. Read `process.env[webhookEnvVar]` (or platform equivalent). If the value is missing, undefined, or the empty string after a `.trim()`, reject with the literal message `"slack-transport: webhook env var {webhookEnvVar} not set; configure a secret_ref binding at company scope"` and exit before any network call. The literal phrase `webhook env var` MUST appear so QA2's T2 can pattern-match it. The resolved URL value MUST NOT be logged, returned in error results, or passed anywhere except into the POST in step 3.

3. **POST attempt.** Initialize an attempt counter (`attempts = 1` for the first attempt; increments per retry). Issue a single HTTPS POST:
   - URL: the resolved webhook URL from step 2.
   - Headers: `Content-Type: application/json` only.
   - Body: `JSON.stringify(message)` from step 1.
   - Timeout: 10 seconds per attempt (network-error class on timeout — see step 4).
   Capture the HTTP status code, the response body (as a UTF-8 string; Slack returns plain text, not JSON), and the `Retry-After` response header (for 429 only).

4. **Classify response.** Apply the four-class policy from BA research-summary §1.5 in this exact order:
   - **HTTP 200.** Success. Slack's documented body is the literal three-character string `"ok"` per [Slack's incoming-webhook reference](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks). Return `{ ok: true, status: 200, attempts }` and exit. Body content other than `"ok"` on a 200 is unexpected; record it in a `note` field (`{ ok: true, status: 200, attempts, note: "unexpected 200 body: <body>" }`) but still treat as success — Slack does not retry-recover from a 200, and a duplicate POST creates a duplicate message. Do NOT retry on 200 under any condition.
   - **HTTP 4xx (any 400-class status).** Permanent failure. Slack's response body is the named-error code (e.g., `invalid_blocks`, `invalid_payload`, `channel_not_found`, `no_active_hooks`, `invalid_token`, `action_prohibited`, `channel_is_archived`, `no_service` per BA §1.4). Return `{ ok: false, status: <code>, error: <body string verbatim>, attempts }` and exit. Do NOT retry. The literal phrase the caller pattern-matches on a permanent failure is the named-error string in the `error` field — this is the "named-error surface" the BA research §1.5 calls out.
   - **HTTP 429 (rate-limited).** Read the `Retry-After` response header. Parse as a non-negative integer (seconds); if missing or unparseable, default to 1 second; if >60 seconds, abort retries and return `{ ok: false, status: 429, error: "retry_after_too_large", retryAfter: <parsed value>, attempts }`. If retry budget remains (i.e., `attempts < 4`, so we have used at most 3 of 3 retries), sleep the resolved `Retry-After` seconds, set `attempts = attempts + 1`, return to step 3. Otherwise (retry budget exhausted), return `{ ok: false, status: 429, error: "retries_exhausted", attempts }` and exit.
   - **HTTP 5xx, network error, or per-attempt timeout.** If retry budget remains (`attempts < 4`), sleep `2^(attempts-1)` seconds — i.e., 1s before retry 1, 2s before retry 2, 4s before retry 3 — set `attempts = attempts + 1`, return to step 3. Otherwise (retry budget exhausted), return `{ ok: false, status: <code or null for network errors>, error: "retries_exhausted", attempts }` and exit. The literal phrase `retries_exhausted` MUST appear in the `error` field so QA2's T6/T7 can pattern-match it.

5. **Result envelope.** The skill's only output is the result object returned in step 4. The result envelope is the closed shape `{ ok: boolean, status: number | null, attempts: number, error?: string, retryAfter?: number, note?: string }` — additional top-level fields are gate-failing on the quality bar's "closed shape" rule. The result MUST NOT include the resolved webhook URL, MUST NOT include the message body, and MUST NOT include any agent / company / project identifier — this skill does not know about those concepts.

## Quality bar

- Every reject path uses one of the literal phrases enumerated above (`message argument is required`, `webhook env var`, `retries_exhausted`, `retry_after_too_large`). QA2 pattern-matches these in T1–T7; rephrasing collapses the test surface.
- The retry cap is exactly **3** (so at most 4 wire attempts per invocation: the initial attempt plus 3 retries). The retry decision is per-attempt, not global — a 429 can follow a 5xx, in which case the second retry uses the 429 branch's `Retry-After` (not the prior 5xx branch's exponential backoff). The `attempts` counter is shared across branches.
- The 5xx / network exponential backoff is exactly the sequence **1s, 2s, 4s** (i.e., `2^(attempts-1)` seconds before retry attempts 1, 2, 3). Jitter is NOT introduced in v1 — the sequence is deterministic so QA2 can assert on elapsed time in T6.
- The four response classes are evaluated in the order `200 → 4xx → 429 → 5xx/network`. `429` is evaluated before generic 5xx specifically because Slack documents 429 as the rate-limit class even though the status code is in the 4xx range; the named-error 4xx branch handles 4xx OTHER than 429. A 429 response that also carries a Slack named-error in its body is treated as 429 (rate-limit retry), NOT as a permanent named-error failure.
- The skill body contains zero Slack message composition logic. It does NOT read or transform the `message` argument's `blocks`, `attachments`, or `text` fields beyond `JSON.stringify` for transport. Per-block character limits, header truncation, mrkdwn escaping, channel routing, and any block-kit shape decisions are upstream concerns and are gate-failing if they appear in this skill body.
- The skill body contains zero Paperclip API calls. No `GET /api/issues`, no `POST /api/issues/:id/comments`, no `/api/agents/me`, no `/api/companies/:id/secrets` direct value reads. The webhook URL arrives via env-var injection only; the skill does NOT resolve secrets via the Paperclip API at run time.
- Outbound HTTPS uses anonymous transport — no `Authorization` header is added on the webhook POST. Slack's webhook token in the URL path is the only credential, and Paperclip's `secret_ref` binding mechanism is the only injection path.
- The resolved webhook URL is touched exactly once per attempt (the POST in step 3). It is NOT logged, NOT included in any return field, NOT echoed to stdout/stderr, and NOT passed into any other call. Treating the URL as transient secret material is gate-failing if violated.
- The result envelope is the closed shape `{ ok, status, attempts, error?, retryAfter?, note? }` — adding extra top-level fields (e.g., `webhookUrl`, `messageBody`, `responseHeaders`, `slackTraceId`) is gate-failing on the closed-shape rule. Future v1.1 fields land via a documented schema bump under [COPAAA-103](/COPAAA/issues/COPAAA-103) follow-up, not via silent extension.
- A duplicate POST on a 200 response is gate-failing (Slack webhooks are not idempotent — a retry on 200 produces a duplicate Slack message). Step 4's "200 short-circuit" is the policy that prevents this; any code path that retries after observing a 200 is gate-failing on T3.
- Per-skill README row update is mandatory at Code Gate per the [COPAAA-19](/COPAAA/issues/COPAAA-19) B4 rule. The slack-transport row in the repo `README.md` lands in the same coding commit as this SKILL.md and is reaffirmed in the Code Gate evidence package per [per_skill_readme_row_rule.md](per_skill_readme_row_rule.md).
- No `references/` subdirectory is created for slack-transport. The skill body is small enough to live in a single SKILL.md per BA research summary §4 (skill structural conventions); future operations or larger response-classification tables would justify extraction, but v1 has no such surface.
