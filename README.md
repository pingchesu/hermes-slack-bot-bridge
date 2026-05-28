# Hermes Slack Bot Bridge

English | [繁體中文](README.zh-TW.md)

Bot-to-bot ingress for Hermes over Slack.

Use this when an external system — GitHub Actions, a CI runner, a webhook relay, or another Slack app — cannot call the Hermes gateway directly, but can post a structured message in Slack. The plugin validates that message, deduplicates it by `request_id`, and rewrites the JSON envelope into a normal Hermes prompt.

> First-time setup: see [`docs/INSTALL.md`](docs/INSTALL.md) for the full operator guide.
>
> The guide covers Slack App creation, bot token scopes, how to get Slack bot `user_id`, and the exact Hermes `.env` / `config.yaml` settings.

## What problem does this solve?

Hermes normally treats Slack bot messages differently from human messages. This plugin lets a known relay bot wake Hermes in a dedicated bridge channel without patching Hermes core and without exposing Hermes to the public internet.

Flow:

```text
External system / CI / webhook relay
  -> Slack chat.postMessage
  -> #hermes-bridge channel
  -> @Hermes hermes-bridge + JSON envelope
  -> slack-bot-bridge validates / deduplicates / rewrites
  -> normal Hermes dispatch
```

## Quick setup

First collect these Slack IDs. The full lookup steps are in [`docs/INSTALL.md`](docs/INSTALL.md).

| Value | Shape | Where it is used |
| --- | --- | --- |
| Bridge channel ID | `C...` | `HERMES_SLACK_BRIDGE_CHANNEL` |
| Hermes bot user ID | `U...` | `<@U...>` mention in the Slack message |
| Relay bot user ID | `U...` | `SLACK_ALLOWED_USERS`, so Hermes authorization accepts the relay bot |
| Relay bot ID | `B...` | `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS` |
| Relay app ID | `A...` | `HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS` |
| Workspace / team ID | `T...` | `HERMES_SLACK_BRIDGE_ALLOWED_TEAMS` |

Install the plugin:

```bash
HERMES_SLACK_BRIDGE_CHANNEL=C0123456789 \
  hermes plugins install pingchesu/hermes-slack-bot-bridge --enable
```

Configure the Hermes Slack adapter to accept bot messages only when Hermes is mentioned:

```yaml
# ~/.hermes/config.yaml
slack:
  allow_bots: mentions
  strict_mention: true
```

Configure Hermes environment variables:

```bash
# ~/.hermes/.env
# Keep existing human users, then add the relay bot's U... user_id.
SLACK_ALLOWED_USERS=U_YOUR_USER,U_RELAY_BOT_USER

HERMES_SLACK_BRIDGE_CHANNEL=C0123456789
HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS=B0AAAAAAA
HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS=A0BBBBBBB
HERMES_SLACK_BRIDGE_ALLOWED_TEAMS=T0CCCCCCC
HERMES_SLACK_BRIDGE_HMAC_SECRET=replace-with-a-long-random-secret
```

Restart the Hermes gateway:

```bash
hermes gateway restart
```

## Slack message format

The relay bot posts a message like this to the bridge channel:

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

Important details:

- `<@U_HERMES_BOT>` must use the Hermes bot's Slack **user ID** (`U...`), not its bot ID (`B...`).
- `hermes-bridge` is the required marker token.
- Put the JSON envelope in a fenced <code>```json</code> block.
- `request_id` is used for deduplication; repeated IDs are ignored inside the dedup TTL.
- If `HERMES_SLACK_BRIDGE_HMAC_SECRET` is set, every envelope must include a valid `signature`.

## Getting Slack bot user IDs

The most reliable method is Slack `auth.test` with the relevant bot token:

```bash
curl -sS -H "Authorization: Bearer <relay-bot-token>" \
  https://slack.com/api/auth.test | jq .
```

The response includes:

```json
{
  "team_id": "T0123456789",
  "user_id": "U0123456789",
  "bot_id": "B0123456789"
}
```

Use `user_id` (`U...`) for `SLACK_ALLOWED_USERS` and Slack mentions. Use `bot_id` (`B...`) for `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS`.

## More documentation

- [`README.zh-TW.md`](README.zh-TW.md) — Traditional Chinese README.
- [`docs/INSTALL.md`](docs/INSTALL.md) — Full installation guide, including Slack ID lookup, smoke tests, and a GitHub Actions relay example.
- [`after-install.md`](after-install.md) — Short checklist shown after `hermes plugins install`.

## Security model

- **Bridge channel allowlist is mandatory.** Without `HERMES_SLACK_BRIDGE_CHANNEL`, the plugin ignores every message.
- **Sender allowlists are recommended for production.** Restrict accepted senders with the relay bot `bot_id` or app `app_id`.
- **HMAC is strongly recommended for production.** Without HMAC, another member of the bridge channel could hand-write a JSON envelope.
- **Hermes Slack authorization still runs.** The relay bot's `user_id` (`U...`) must be in `SLACK_ALLOWED_USERS`, otherwise the gateway can reject the rewritten prompt.
- **Free-form bot text is ignored.** The plugin requires a mention, the `hermes-bridge` marker, and a JSON envelope.
