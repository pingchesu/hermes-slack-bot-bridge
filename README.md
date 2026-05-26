# Hermes Slack Bot Bridge

Bot-to-bot ingress for Hermes instances that live behind a firewall. An
external bot — a GitHub Actions workflow, a webhook relay, a CI runner —
posts a tagged JSON envelope into one or more dedicated Slack channels, and
Hermes treats it as if a human had typed the prompt themselves.

The plugin **does not** patch the Slack adapter or the gateway runner. It
hooks `pre_gateway_dispatch`, validates the envelope, deduplicates by
`request_id`, and rewrites the Slack message into a canonical prompt before
the normal dispatch path takes over.

## Recommended Slack path

Use **Slack Web API `chat.postMessage` from a relay bot** for production
bot-to-bot traffic:

```text
external service / bot
  -> Slack Web API chat.postMessage
  -> dedicated Slack bridge channel
  -> Hermes Slack adapter
  -> slack-bot-bridge plugin
  -> normal Hermes dispatch
```

Slack Incoming Webhooks can post into the channel, but they are not the
recommended path for this plugin. Incoming-webhook events may arrive with a
`bot_id` but no normal bot `user_id` / `team` identity; the plugin can accept
the envelope, but Hermes' later Slack authorization/dispatch path may still
not produce an agent response. If you need reliable request/response behavior,
use `chat.postMessage` with a Bot User OAuth Token that has `chat:write`.

## Install

```bash
hermes plugins install pingchesu/hermes-slack-bot-bridge --enable
```

This installs into `~/.hermes/plugins/slack-bot-bridge/`, not into the
`hermes-agent` source tree. Update later with:

```bash
hermes plugins update slack-bot-bridge
```

## Slack setup checklist

Before changing Hermes config, set up or select a Slack App that will act as
the **relay bot**.

1. In Slack App settings, open **OAuth & Permissions**.
2. Add Bot Token Scopes:
   - `chat:write` — required for `chat.postMessage`.
   - `chat:write.public` — optional; only needed if the bot must post to
     public channels it has not joined.
   - `incoming-webhook` alone is **not enough** for `chat.postMessage`. Slack
     will return `missing_scope` with `needed: chat:write:bot` or
     `needed: chat:write`.
3. Click **Reinstall to Workspace** after scope changes.
4. Copy the **Bot User OAuth Token** (`xoxb-...`) for the relay bot and store
   it as a secret. Do not paste it into Slack or commit it to git.
5. Create or choose a dedicated bridge channel, then invite both bots:
   - the Hermes Slack bot
   - the relay bot that will call `chat.postMessage`

## Configure Hermes

### 1. Allow Slack bot messages to reach the plugin

In `~/.hermes/config.yaml`, let relay bot messages through when they mention
Hermes:

```yaml
slack:
  allow_bots: mentions
  strict_mention: true
```

`allow_bots: all` also works in a dedicated automation channel, but `mentions`
is safer. Without `allow_bots: mentions` or an equivalent setting, the Slack
adapter can drop bot messages before the plugin sees them.

Do **not** set `slack.allowed_channels` to only the bridge channel if this
Hermes bot already serves other Slack channels. `allowed_channels` is a global
Slack-adapter allowlist and would block Hermes in every channel not listed.
Use `HERMES_SLACK_BRIDGE_CHANNEL` to limit bridge ingress instead.

### 2. Get the required Slack IDs

You need three IDs:

| ID | Where used | How to get it |
| --- | --- | --- |
| Hermes bot user ID (`U...`) | Mention Hermes in the Slack message text: `<@U_HERMES_BOT>` | Run `auth.test` with Hermes' Slack bot token, or copy from Slack profile/app details. |
| Relay bot user ID (`U...`) | Add to `SLACK_ALLOWED_USERS` when Hermes Slack authorization is enabled | Run `auth.test` with the relay bot token. |
| Relay bot ID (`B...`) | Add to `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS` | Run `auth.test` with the relay bot token. |

Example `auth.test` for the **relay** bot:

```bash
export RELAY_SLACK_BOT_TOKEN='<relay-bot-token>'

curl -sS https://slack.com/api/auth.test \
  -H "Authorization: Bearer ${RELAY_SLACK_BOT_TOKEN}"
```

Successful output includes fields like:

```json
{
  "ok": true,
  "user_id": "U_RELAY_BOT",
  "bot_id": "B_RELAY_BOT",
  "team_id": "T_WORKSPACE"
}
```

Example `auth.test` for the **Hermes** bot, if its token is in Hermes' env as
`SLACK_BOT_TOKEN`:

```bash
set -a
. ~/.hermes/.env
set +a

curl -sS https://slack.com/api/auth.test \
  -H "Authorization: Bearer ${SLACK_BOT_TOKEN}"
```

Use the returned Hermes `user_id` as `HERMES_USER_ID` in test payloads.

### 3. Configure bridge env

In `~/.hermes/.env`:

```bash
# Dedicated Slack channel(s) where bridge messages are accepted.
# Comma-separated for multiple channels.
HERMES_SLACK_BRIDGE_CHANNEL=C0123456789

# Relay bot identity from auth.test. Production should set at least this
# or HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS.
HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS=B_RELAY_BOT

# Optional app allowlist if you know the relay app id from Slack events/logs.
HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS=A_RELAY_APP

# Optional HMAC signature on every envelope; strongly recommended for prod.
HERMES_SLACK_BRIDGE_HMAC_SECRET=use-a-real-secret-here

# Optional: override the 24-hour dedup window (0 = never expire entries).
HERMES_SLACK_BRIDGE_DEDUP_TTL_SECONDS=86400
```

Also make sure the relay bot **user ID** is allowed by Hermes' Slack auth, if
`SLACK_ALLOWED_USERS` is enabled:

```bash
SLACK_ALLOWED_USERS=U_OWNER,U_RELAY_BOT
```

Notes:

- `HERMES_SLACK_BRIDGE_CHANNEL` is required; the plugin is inert without it.
- For quick bring-up, you may temporarily use channel-only gating, but
  production should configure a bot/app allowlist plus HMAC.
- Do not set `HERMES_SLACK_BRIDGE_ALLOWED_TEAMS` until you have verified the
  Slack event actually carries a `team` value. Incoming-webhook events may have
  an empty `team`, which causes team allowlist checks to reject the message.

### 4. Restart Hermes gateway

After config/env changes:

```bash
hermes gateway restart
```

## Wire format

A relay posts a Slack message like:

````text
<@U_HERMES_BOT> hermes-bridge
```json
{
  "request_id": "ci-pr-4242-attempt-1",
  "actor": "github-actions",
  "prompt": "Triage failures in PR #4242 — focus on the auth tests.",
  "metadata": {"repo": "org/repo", "pr": 4242},
  "signature": "f5c1...optional-hex-hmac-sha256"
}
```
````

- `<@U_HERMES_BOT>` is the standard Slack mention of your Hermes bot user.
- `hermes-bridge` is the literal marker token (case-insensitive).
- The envelope MUST sit inside a fenced ` ```json ... ``` ` block; the plugin
  also accepts a bare `{ ... }` for clients that strip code fences.
- The mention must be in top-level Slack message `text`. Do not rely only on
  Block Kit text for the Hermes mention.

Field rules:

| field | required | shape |
| --- | --- | --- |
| `request_id` | yes | opaque string `[A-Za-z0-9_\-\.:]{1,128}`, used for dedup |
| `prompt` | yes | text — up to 32 000 characters |
| `actor` | no | opaque string, same character set as `request_id` |
| `metadata` | no | JSON object — passed through to the agent inside the rewritten prompt |
| `signature` | depends | hex HMAC-SHA256 over `request_id|actor|prompt|canonical_metadata`; required iff secret set |

## Smoke test with `chat.postMessage`

Set these from your Slack App and Hermes setup:

```bash
export RELAY_SLACK_BOT_TOKEN='<relay-bot-token>'
export HERMES_BRIDGE_CHANNEL='C0123456789'
export HERMES_USER_ID='U_HERMES_BOT'
```

Post a test envelope:

````bash
curl -sS https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer ${RELAY_SLACK_BOT_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data "$(jq -nc \
    --arg channel "$HERMES_BRIDGE_CHANNEL" \
    --arg text '<@'"$HERMES_USER_ID"'> hermes-bridge
```json
{"request_id":"chatpost-smoke-'"$(date +%s)"'","actor":"chat.postMessage-bot","prompt":"請回覆：chat.postMessage bridge test OK","metadata":{"source":"chat.postMessage"}}
```' \
    '{channel:$channel,text:$text}')"
````

Expected Slack API response:

```json
{"ok": true, "channel": "C0123456789", "ts": "..."}
```

Then verify Hermes gateway logs show both inbound and response lines, e.g.:

```text
inbound message: platform=slack user=<relay bot name> chat=C0123456789 msg='@Hermes ... hermes-bridge ...'
slack-bot-bridge: accepted envelope request_id=chatpost-smoke-... actor=chat.postMessage-bot
response ready: platform=slack chat=C0123456789 ...
```

## Common Slack errors

| Error | Meaning | Fix |
| --- | --- | --- |
| `not_authed` | The token variable is empty or invalid. | Check the shell variable used in `Authorization: Bearer ...`; run `auth.test`. |
| `missing_scope`, `provided: incoming-webhook` | You are using a token/app that only has incoming-webhook scope. | Add `chat:write`, reinstall the Slack App, and use the Bot User OAuth Token. |
| `channel_not_found` | Bot cannot see/post to the channel. | Invite the relay bot to the channel, or add `chat:write.public` for public channels. |
| Hermes plugin logs `sender-not-allowlisted` | Sender bot/app/team did not match plugin allowlist. | Add the relay bot `bot_id`/app id to bridge allowlist, or remove a stale team allowlist. |
| Plugin logs `accepted envelope` but no Hermes response | Message passed plugin validation but normal Hermes Slack auth/dispatch did not complete. | Confirm relay bot `user_id` is in `SLACK_ALLOWED_USERS` and gateway was restarted. |
| Plugin logs `duplicate-request-id` | Same `request_id` already processed in dedup window. | Send a new `request_id` for each logical request. |

## Incoming Webhook caveat

Slack Incoming Webhook is useful for writing into Slack, but it is not the
preferred request/response ingress path here.

Known behavior:

- It may produce Slack events with `bot_id` but no normal `user_id` / `team`.
- `HERMES_SLACK_BRIDGE_ALLOWED_TEAMS` can reject such events because `team` is
  empty.
- The plugin may log `accepted envelope`, while the normal Hermes Slack
  authorization/dispatch path still does not produce a reply.
- A token/app with only `incoming-webhook` scope cannot call
  `chat.postMessage`.

If you must test Incoming Webhook anyway:

````bash
export SLACK_INCOMING_WEBHOOK_URL='https://hooks.slack.com/services/...'
export HERMES_USER_ID='U_HERMES_BOT'

curl -sS -X POST "$SLACK_INCOMING_WEBHOOK_URL" \
  -H 'Content-Type: application/json' \
  --data "$(jq -nc \
    --arg text '<@'"$HERMES_USER_ID"'> hermes-bridge
```json
{"request_id":"webhook-smoke-'"$(date +%s)"'","actor":"slack-incoming-webhook","prompt":"請回覆：incoming webhook bridge test OK","metadata":{"source":"incoming-webhook"}}
```' \
    '{text:$text}')"
````

For reliable bot-to-bot triggering, use `chat.postMessage` instead.

## What Hermes actually sees

For a valid envelope, the agent receives:

```text
[slack-bot-bridge]
request_id: ci-pr-4242-attempt-1
actor: github-actions
metadata: {"pr": 4242, "repo": "org/repo"}

Triage failures in PR #4242 — focus on the auth tests.
```

## GitHub Actions example

````yaml
# .github/workflows/hermes-bridge.yml
name: Notify Hermes
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Build envelope and post to Slack
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
          BRIDGE_CHANNEL: ${{ vars.HERMES_BRIDGE_CHANNEL }}
          HERMES_USER_ID: ${{ vars.HERMES_USER_ID }}
          BRIDGE_HMAC_SECRET: ${{ secrets.HERMES_SLACK_BRIDGE_HMAC_SECRET }}
        run: |
          set -euo pipefail
          REQUEST_ID="pr-${{ github.event.pull_request.number }}-sha-${{ github.event.pull_request.head.sha }}"
          ACTOR="github-actions"
          PROMPT="Triage failures in PR #${{ github.event.pull_request.number }}."
          METADATA=$(jq -S -nc --arg repo "${{ github.repository }}" '{"pr": ${{ github.event.pull_request.number }}, "repo": $repo}')
          # signature input is the same canonical form the plugin computes
          SIG=$(printf '%s|%s|%s|%s' "$REQUEST_ID" "$ACTOR" "$PROMPT" "$METADATA" \
                | openssl dgst -sha256 -hmac "$BRIDGE_HMAC_SECRET" \
                | awk '{print $2}')
          BODY=$(jq -nc \
            --arg request_id "$REQUEST_ID" \
            --arg actor      "$ACTOR" \
            --arg prompt     "$PROMPT" \
            --arg signature  "$SIG" \
            --argjson metadata "$METADATA" \
            '{request_id:$request_id, actor:$actor, prompt:$prompt, metadata:$metadata, signature:$signature}')
          # Slack message text — fenced JSON, mention Hermes, include marker.
          TEXT=$(printf '<@%s> hermes-bridge\n```json\n%s\n```\n' "$HERMES_USER_ID" "$BODY")
          curl -fsS -X POST https://slack.com/api/chat.postMessage \
            -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
            -H "Content-Type: application/json; charset=utf-8" \
            --data "$(jq -nc --arg ch "$BRIDGE_CHANNEL" --arg t "$TEXT" '{channel:$ch, text:$t}')"
````

## Security model

- **Channel allowlist** is mandatory — the plugin is inert without
  `HERMES_SLACK_BRIDGE_CHANNEL`.
- **Identifier allowlists** (bot/app/team) narrow the set of senders the
  plugin will accept inside the bridge channel(s). Configure at least one app
  or bot id for production; team allowlists are an additional scope gate.
- **HMAC** (optional but strongly recommended for production) makes the
  envelope tamper-evident; without it any member of a bridge channel could
  craft a payload.
- **No free-form text.** The plugin only accepts a structured envelope —
  arbitrary bot chatter in the bridge channel is ignored.
- **Dedup** by `request_id` for 24 hours by default — Slack retries are
  effectively idempotent.
- **Auth still runs.** The plugin returns a `rewrite` action; the gateway's
  normal `SLACK_ALLOWED_USERS` / pairing check still applies to the sending
  bot's Slack user id. Add the relay bot to that allowlist explicitly.
- **Token hygiene.** If a bot token is pasted into Slack, logs, or commits,
  revoke/rotate it before using that app in production.
