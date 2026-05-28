# Install Hermes Slack Bot Bridge

這份文件從零開始說明怎麼把 Slack bot-to-bot bridge 裝起來。

目標結果：

1. 你有一個專用 Slack channel，例如 `#hermes-bridge`。
2. Hermes bot 和 relay bot 都在這個 channel 裡。
3. Relay bot 用 Slack API `chat.postMessage` 發訊息。
4. Hermes 只接受符合 allowlist / HMAC / JSON envelope 的 bot 訊息。

---

## 0. 你需要先準備什麼

- 一個已經可以在 Slack 收訊息的 Hermes。
- Hermes CLI 可用：

  ```bash
  hermes --help
  ```

- Slack workspace 管理權限，或至少能建立 / 安裝 Slack App。
- 一個 relay bot 的 Slack Bot User OAuth Token，長相是 `xoxb-...`。

如果你已經有 relay bot，可以跳到 [2. 取得所有 Slack ID](#2-取得所有-slack-id)。

---

## 1. 建立 relay Slack App / Bot

到 <https://api.slack.com/apps>：

1. 點 **Create New App**。
2. 選 **From scratch**。
3. App name 例如：`Hermes Relay Bot`。
4. 選你的 workspace。
5. 進入 **OAuth & Permissions**。
6. 在 **Bot Token Scopes** 加上：

   | Scope | 用途 |
   | --- | --- |
   | `chat:write` | relay bot 發訊息到 bridge channel |

   如果你想用 API 查 channel history 來確認 `bot_id` / `app_id`，可以暫時加：

   | Optional scope | 用途 |
   | --- | --- |
   | `channels:history` | 查 public channel 最新訊息 |
   | `groups:history` | 查 private channel 最新訊息 |

7. 點 **Install to Workspace**。
8. 複製 **Bot User OAuth Token**，長相：`xoxb-...`。
9. 到 Slack 建立專用 channel，例如 `#hermes-bridge`。
10. 把 Hermes bot 和 relay bot 都 invite 進 channel：

    ```text
    /invite @Hermes
    /invite @Hermes Relay Bot
    ```

> 不要用 Incoming Webhook 當 relay。Webhook 可以送訊息，但不一定有穩定的 bot user authorization path；這個 plugin 主要設計給 Slack App bot token + `chat.postMessage`。

---

## 2. 取得所有 Slack ID

這一步最重要。很多設定失敗都是因為把 `U...`、`B...`、`A...` 混用。

### 2.1 ID 對照表

| 名稱 | 長相 | 怎麼拿 | 用在哪裡 |
| --- | --- | --- | --- |
| Bridge channel ID | `C...` | Slack channel details / URL / API | `HERMES_SLACK_BRIDGE_CHANNEL` |
| Hermes bot user ID | `U...` | Hermes bot token `auth.test` 或 Slack profile | Relay 訊息裡 `<@U...>` |
| Relay bot user ID | `U...` | Relay bot token `auth.test` | `SLACK_ALLOWED_USERS` |
| Relay bot ID | `B...` | Relay bot token `auth.test` 或 message history | `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS` |
| Relay app ID | `A...` | Slack App Basic Information 或 message history | `HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS` |
| Team ID | `T...` | `auth.test` | `HERMES_SLACK_BRIDGE_ALLOWED_TEAMS` |

### 2.2 用 Slack API 拿 bot user ID（推薦）

用 bot token 打 `auth.test`。這是最不容易搞錯的方法。

Relay bot：

```bash
export RELAY_SLACK_BOT_TOKEN='xoxb-your-relay-token'

curl -sS -H "Authorization: Bearer $RELAY_SLACK_BOT_TOKEN" \
  https://slack.com/api/auth.test | jq .
```

你會看到類似：

```json
{
  "ok": true,
  "url": "https://example.slack.com/",
  "team": "Example",
  "user": "hermes-relay",
  "team_id": "T0123456789",
  "user_id": "U0123456789",
  "bot_id": "B0123456789"
}
```

請記住：

- `user_id` (`U...`)：要加到 Hermes 的 `SLACK_ALLOWED_USERS`。
- `bot_id` (`B...`)：可以填到 `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS`。
- `team_id` (`T...`)：可以填到 `HERMES_SLACK_BRIDGE_ALLOWED_TEAMS`。

Hermes bot 的 user ID 也一樣用 Hermes bot token 查：

```bash
export HERMES_SLACK_BOT_TOKEN='xoxb-your-hermes-bot-token'

curl -sS -H "Authorization: Bearer $HERMES_SLACK_BOT_TOKEN" \
  https://slack.com/api/auth.test | jq -r '.user_id'
```

這個回傳的 `U...` 會放在 relay 訊息的 mention 裡：

```text
<@U_HERMES_BOT> hermes-bridge
```

### 2.3 用 Slack UI 拿 user ID

如果你沒有 token，也可以用 UI：

1. 在 Slack 找到 bot 的 profile。
2. 點 profile / app details。
3. 找 **More** / `...`。
4. 點 **Copy member ID**。
5. 複製到的應該是 `U...`。

注意：Slack UI 有時只顯示 app 資訊，不容易找到 bot user ID。找不到時，請改用 `auth.test`。

### 2.4 取得 channel ID

方法 A：Slack UI

1. 打開 bridge channel。
2. 點 channel 名稱。
3. 最下面找 **Channel ID**，或點 **Copy channel ID**。

方法 B：Slack URL

Slack channel URL 通常長這樣：

```text
https://app.slack.com/client/T0123456789/C0123456789
```

第二段 `C0123456789` 就是 channel ID。

### 2.5 取得 relay app ID (`A...`)

方法 A：Slack App 後台

1. 到 <https://api.slack.com/apps>。
2. 點你的 relay app。
3. 進 **Basic Information**。
4. 找 **App ID**，長相 `A...`。

方法 B：先發一則測試訊息，再查 latest message

```bash
export BRIDGE_CHANNEL='C0123456789'
export HERMES_USER_ID='U_HERMES_BOT'

curl -fsS -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $RELAY_SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data "$(jq -nc \
    --arg ch "$BRIDGE_CHANNEL" \
    --arg text "<@$HERMES_USER_ID> hermes-bridge test-id-lookup" \
    '{channel:$ch, text:$text}')"

curl -sS -H "Authorization: Bearer $RELAY_SLACK_BOT_TOKEN" \
  "https://slack.com/api/conversations.history?channel=$BRIDGE_CHANNEL&limit=1" \
  | jq '.messages[0] | {user, bot_id, app_id, team}'
```

---

## 3. 安裝 plugin

建議先拿到 channel ID 後再安裝，因為 plugin manifest 會要求 `HERMES_SLACK_BRIDGE_CHANNEL`。

```bash
HERMES_SLACK_BRIDGE_CHANNEL=C0123456789 \
  hermes plugins install pingchesu/hermes-slack-bot-bridge --enable
```

更新 plugin：

```bash
hermes plugins update slack-bot-bridge
```

確認 plugin 已啟用：

```bash
hermes plugins list
```

---

## 4. 設定 Hermes Slack adapter

Hermes Slack adapter 預設通常會丟掉 bot message。Bridge 需要讓「有 mention Hermes 的 bot message」通過 adapter。

在 `~/.hermes/config.yaml` 加上或確認：

```yaml
slack:
  allow_bots: mentions
  strict_mention: true
```

也可以用環境變數：

```bash
SLACK_ALLOW_BOTS=mentions
SLACK_STRICT_MENTION=true
```

不要為了這個 plugin 把 `slack.allowed_channels` 設成只有 bridge channel，除非這個 Hermes bot 真的只服務 bridge channel。`slack.allowed_channels` 是整個 Slack adapter 的全域 allowlist，設錯會讓 Hermes 在其他 channel 全部失效。Bridge 的入口限制請用 `HERMES_SLACK_BRIDGE_CHANNEL`。

---

## 5. 設定 Hermes `.env`

編輯 `~/.hermes/.env`：

```bash
# 既有的人類使用者要保留，再加上 relay bot 的 U... user_id。
# 這裡一定要填 U...，不是 B...。
SLACK_ALLOWED_USERS=U_YOUR_USER,U_RELAY_BOT_USER

# 必填：只接受這些 channel 的 bridge envelope。
HERMES_SLACK_BRIDGE_CHANNEL=C0123456789

# 正式環境建議至少填 bot_id 或 app_id 其中一種。
HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS=B0123456789
HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS=A0123456789
HERMES_SLACK_BRIDGE_ALLOWED_TEAMS=T0123456789

# 正式環境強烈建議開 HMAC。
HERMES_SLACK_BRIDGE_HMAC_SECRET=replace-with-a-long-random-secret

# 可選：request_id 去重保留秒數，預設 86400。
HERMES_SLACK_BRIDGE_DEDUP_TTL_SECONDS=86400
```

產生 secret 範例：

```bash
python3 - <<'PY'
import secrets
print(secrets.token_urlsafe(32))
PY
```

---

## 6. 重啟 Hermes gateway

```bash
hermes gateway restart
```

如果你的 Hermes 是用 systemd / supervisor / Docker 跑，請用你自己的服務重啟方式。重點是讓新的 config / env / plugin 被 gateway 載入。

---

## 7. 發一則測試 bridge message

### 7.1 不開 HMAC 的最小測試

只建議 quick bring-up 使用；正式環境請看下一節。

```bash
export RELAY_SLACK_BOT_TOKEN='xoxb-your-relay-token'
export BRIDGE_CHANNEL='C0123456789'
export HERMES_USER_ID='U_HERMES_BOT'

BODY=$(jq -nc \
  --arg request_id "manual-test-$(date +%s)" \
  --arg actor "manual-test" \
  --arg prompt "Reply with: slack bot bridge smoke test ok" \
  '{request_id:$request_id, actor:$actor, prompt:$prompt, metadata:{source:"manual"}}')

TEXT=$(printf '<@%s> hermes-bridge\n```json\n%s\n```\n' "$HERMES_USER_ID" "$BODY")

curl -fsS -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $RELAY_SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data "$(jq -nc --arg ch "$BRIDGE_CHANNEL" --arg text "$TEXT" '{channel:$ch, text:$text}')" \
  | jq .
```

### 7.2 有 HMAC 的正式測試

```bash
export RELAY_SLACK_BOT_TOKEN='xoxb-your-relay-token'
export BRIDGE_CHANNEL='C0123456789'
export HERMES_USER_ID='U_HERMES_BOT'
export BRIDGE_HMAC_SECRET='same-value-as-HERMES_SLACK_BRIDGE_HMAC_SECRET'

REQUEST_ID="manual-test-$(date +%s)"
ACTOR="manual-test"
PROMPT="Reply with: slack bot bridge signed smoke test ok"
METADATA=$(jq -S -nc '{source:"manual"}')

SIG=$(printf '%s|%s|%s|%s' "$REQUEST_ID" "$ACTOR" "$PROMPT" "$METADATA" \
  | openssl dgst -sha256 -hmac "$BRIDGE_HMAC_SECRET" \
  | awk '{print $2}')

BODY=$(jq -nc \
  --arg request_id "$REQUEST_ID" \
  --arg actor "$ACTOR" \
  --arg prompt "$PROMPT" \
  --arg signature "$SIG" \
  --argjson metadata "$METADATA" \
  '{request_id:$request_id, actor:$actor, prompt:$prompt, metadata:$metadata, signature:$signature}')

TEXT=$(printf '<@%s> hermes-bridge\n```json\n%s\n```\n' "$HERMES_USER_ID" "$BODY")

curl -fsS -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $RELAY_SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data "$(jq -nc --arg ch "$BRIDGE_CHANNEL" --arg text "$TEXT" '{channel:$ch, text:$text}')" \
  | jq .
```

成功時，你應該會在 Slack 看到 Hermes 回覆。

---

## 8. GitHub Actions relay 範例

Repo secrets / variables：

| Name | 類型 | 值 |
| --- | --- | --- |
| `SLACK_BOT_TOKEN` | Secret | relay bot 的 `xoxb-...` token |
| `HERMES_SLACK_BRIDGE_HMAC_SECRET` | Secret | 與 Hermes `.env` 相同的 HMAC secret |
| `HERMES_BRIDGE_CHANNEL` | Variable | bridge channel ID (`C...`) |
| `HERMES_USER_ID` | Variable | Hermes bot user ID (`U...`) |

Workflow：

```yaml
name: Notify Hermes

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Send bridge envelope to Slack
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
          BRIDGE_CHANNEL: ${{ vars.HERMES_BRIDGE_CHANNEL }}
          HERMES_USER_ID: ${{ vars.HERMES_USER_ID }}
          BRIDGE_HMAC_SECRET: ${{ secrets.HERMES_SLACK_BRIDGE_HMAC_SECRET }}
        run: |
          set -euo pipefail

          REQUEST_ID="pr-${{ github.event.pull_request.number }}-${{ github.event.pull_request.head.sha }}"
          ACTOR="github-actions"
          PROMPT="Triage PR #${{ github.event.pull_request.number }} failures in ${{ github.repository }}."
          METADATA=$(jq -S -nc \
            --arg repo "${{ github.repository }}" \
            --arg pr "${{ github.event.pull_request.number }}" \
            --arg sha "${{ github.event.pull_request.head.sha }}" \
            '{repo:$repo, pr:($pr|tonumber), sha:$sha}')

          SIG=$(printf '%s|%s|%s|%s' "$REQUEST_ID" "$ACTOR" "$PROMPT" "$METADATA" \
            | openssl dgst -sha256 -hmac "$BRIDGE_HMAC_SECRET" \
            | awk '{print $2}')

          BODY=$(jq -nc \
            --arg request_id "$REQUEST_ID" \
            --arg actor "$ACTOR" \
            --arg prompt "$PROMPT" \
            --arg signature "$SIG" \
            --argjson metadata "$METADATA" \
            '{request_id:$request_id, actor:$actor, prompt:$prompt, metadata:$metadata, signature:$signature}')

          TEXT=$(printf '<@%s> hermes-bridge\n```json\n%s\n```\n' "$HERMES_USER_ID" "$BODY")

          curl -fsS -X POST https://slack.com/api/chat.postMessage \
            -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
            -H "Content-Type: application/json; charset=utf-8" \
            --data "$(jq -nc --arg ch "$BRIDGE_CHANNEL" --arg text "$TEXT" '{channel:$ch, text:$text}')"
```

---

## 9. Troubleshooting

### Relay bot 發了訊息，但 Hermes 沒反應

檢查順序：

1. Relay bot 和 Hermes bot 都有在 bridge channel 裡嗎？
2. 訊息有 mention Hermes 嗎？必須是 `<@U_HERMES_BOT>`。
3. `slack.allow_bots: mentions` 或 `SLACK_ALLOW_BOTS=mentions` 有設定嗎？
4. Relay bot 的 `user_id` (`U...`) 有加到 `SLACK_ALLOWED_USERS` 嗎？
5. `HERMES_SLACK_BRIDGE_CHANNEL` 是 channel ID (`C...`) 不是 channel name 嗎？
6. 如果有設定 HMAC，`signature` 是否用同一個 secret 和同一份 canonical metadata 算出來？
7. `request_id` 是否重複？重複會被 dedup。

### 我有 `bot_id`，為什麼 `SLACK_ALLOWED_USERS` 還是不過？

`SLACK_ALLOWED_USERS` 要的是 Slack **user ID** (`U...`)，不是 `bot_id` (`B...`)。

用這個拿 relay bot 的 user ID：

```bash
curl -sS -H "Authorization: Bearer $RELAY_SLACK_BOT_TOKEN" \
  https://slack.com/api/auth.test | jq -r '.user_id'
```

### `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS` 和 `SLACK_ALLOWED_USERS` 差在哪？

- `SLACK_ALLOWED_USERS=U...`：Hermes gateway 自己的 authorization gate。
- `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS=B...`：這個 plugin 自己的 sender allowlist。

兩個 gate 都會跑。正式環境通常兩個都要設定。

### 可以只靠 channel allowlist 嗎？

可以用來 quick bring-up，但不建議正式環境使用。正式環境請至少設定：

```bash
HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS=B...
# or
HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS=A...

HERMES_SLACK_BRIDGE_HMAC_SECRET=...
```

### 可以用 Incoming Webhook 嗎？

不建議。Incoming Webhook 可以送訊息，但 authorization / bot identity 行為不如 Slack App bot token 清楚。建議用 Slack App + `chat.postMessage`。

### 可以用 channel 名稱 `#hermes-bridge` 嗎？

不行。設定值請用 channel ID (`C...`)。

### Mention 裡可以用 bot_id `B...` 嗎？

不行。Slack mention 要用 user ID：

```text
<@U0123456789>
```

不是：

```text
<@B0123456789>
```
