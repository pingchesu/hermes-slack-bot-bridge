# Slack Bot Bridge installed

Next steps for a reliable bot-to-bot setup:

1. Use a Slack relay bot that posts with Web API `chat.postMessage`, not only an Incoming Webhook.
   - Required Slack Bot Token Scope: `chat:write`.
   - Optional: `chat:write.public` if the bot must post to public channels it has not joined.
   - If Slack returns `missing_scope` with `provided: incoming-webhook`, add `chat:write`, reinstall the app, and use the Bot User OAuth Token (`xoxb-...`).
2. Invite both the Hermes bot and the relay bot to the bridge channel.
3. Ensure Slack adapter accepts relay bot messages when Hermes is mentioned:

```yaml
slack:
  allow_bots: mentions
  strict_mention: true
```

Do **not** set `slack.allowed_channels` to only the bridge channel if this Hermes bot already serves other Slack channels. `allowed_channels` is a global Slack-adapter allowlist and would block Hermes in every channel not listed. Use `HERMES_SLACK_BRIDGE_CHANNEL` to limit bridge ingress instead.

4. Get IDs with Slack `auth.test`:
   - Hermes bot `user_id` (`U...`) → mention it in message text as `<@U_HERMES_BOT>`.
   - Relay bot `user_id` (`U...`) → add to `SLACK_ALLOWED_USERS`, if enabled.
   - Relay bot `bot_id` (`B...`) → add to `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS`.

```bash
curl -sS https://slack.com/api/auth.test \
  -H "Authorization: Bearer ${RELAY_SLACK_BOT_TOKEN}"
```

5. Configure `~/.hermes/.env`:

```bash
HERMES_SLACK_BRIDGE_CHANNEL=C0123456789
HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS=B_RELAY_BOT
SLACK_ALLOWED_USERS=U_OWNER,U_RELAY_BOT

# Strongly recommended for production:
HERMES_SLACK_BRIDGE_HMAC_SECRET=use-a-real-secret-here
```

6. Restart the gateway:

```bash
hermes gateway restart
```

7. Smoke test with `chat.postMessage` and make sure the top-level Slack `text` contains both the Hermes mention and `hermes-bridge` marker. Incoming Webhooks may be accepted by the plugin but still fail to produce a Hermes response because their Slack event identity can lack a normal `user_id`.
