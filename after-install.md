# Slack Bot Bridge installed

Next steps:

1. Ensure Slack adapter accepts relay bot messages when Hermes is mentioned:

```yaml
slack:
  allow_bots: mentions
  strict_mention: true
```

Do **not** set `slack.allowed_channels` to only the bridge channel if this Hermes bot already serves other Slack channels. `allowed_channels` is a global Slack-adapter allowlist and would block Hermes in every channel not listed. Use `HERMES_SLACK_BRIDGE_CHANNEL` to limit bridge ingress instead.

2. Add the relay bot Slack user id to `SLACK_ALLOWED_USERS` in `~/.hermes/.env`, if user authorization is enabled.
3. For production, set `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS` or `HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS`, plus `HERMES_SLACK_BRIDGE_HMAC_SECRET`.
4. Restart the gateway: `hermes gateway restart`.
