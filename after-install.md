# Slack Bot Bridge installed

Next steps:

1. Read the full install guide: [`docs/INSTALL.md`](docs/INSTALL.md).
2. Make sure you have the important Slack IDs:

   | Value | Shape | Use |
   | --- | --- | --- |
   | Bridge channel ID | `C...` | `HERMES_SLACK_BRIDGE_CHANNEL` |
   | Hermes bot user ID | `U...` | `<@U...>` mention in relay messages |
   | Relay bot user ID | `U...` | add to `SLACK_ALLOWED_USERS` |
   | Relay bot ID / app ID | `B...` / `A...` | plugin sender allowlist |

   Recommended way to get a bot user ID:

   ```bash
   curl -sS -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
     https://slack.com/api/auth.test | jq -r '.user_id'
   ```

3. Let Hermes' Slack adapter accept bot messages only when Hermes is mentioned:

   ```yaml
   slack:
     allow_bots: mentions
     strict_mention: true
   ```

   Do **not** set `slack.allowed_channels` to only the bridge channel if this Hermes bot already serves other Slack channels. `allowed_channels` is a global Slack-adapter allowlist and would block Hermes in every channel not listed. Use `HERMES_SLACK_BRIDGE_CHANNEL` to limit bridge ingress instead.

4. Add the relay bot Slack **user ID** (`U...`, not `B...`) to `SLACK_ALLOWED_USERS` in `~/.hermes/.env`, if user authorization is enabled.
5. For production, set `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS` or `HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS`, plus `HERMES_SLACK_BRIDGE_HMAC_SECRET`.
6. Restart the gateway: `hermes gateway restart`.
