# Slack Bot Bridge installed

Next steps:

1. Ensure Slack adapter config allows relay bot messages only when Hermes is mentioned:

```yaml
platforms:
  slack:
    extra:
      allow_bots: mentions
      strict_mention: true
      allowed_channels:
        - <bridge-channel-id>
        - <optional-second-bridge-channel-id>
```

2. Add the relay bot Slack user id to `SLACK_ALLOWED_USERS` in `~/.hermes/.env`.
3. For production, set `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS` or `HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS`, plus `HERMES_SLACK_BRIDGE_HMAC_SECRET`.
4. Restart the gateway: `hermes gateway restart`.
