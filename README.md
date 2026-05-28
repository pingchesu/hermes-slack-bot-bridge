# Hermes Slack Bot Bridge

讓「另一個 Slack Bot」把任務丟給 Hermes。

常見用途：GitHub Actions、CI runner、webhook relay、內網外的自動化系統，不能直接打到 Hermes gateway，但可以在 Slack 頻道裡 `@Hermes` 丟一個固定格式的 JSON。這個 plugin 會驗證訊息、去重，然後把 JSON 裡的 `prompt` 轉成 Hermes 可以處理的訊息。

> 第一次安裝請直接看：[`docs/INSTALL.md`](docs/INSTALL.md)
>
> 裡面包含 Slack App 建立、Bot Token scopes、如何拿 Slack bot `user_id`、Hermes `.env` / `config.yaml` 要填什麼、以及測試指令。

## 它解決什麼問題？

Hermes 在 Slack 裡通常只接受真人訊息。這個 plugin 讓「被允許的 bot」也能在指定 bridge channel 裡觸發 Hermes，但不需要改 Hermes core，也不需要讓 Hermes 暴露在 public internet。

流程：

```text
外部系統 / CI / webhook relay
  -> Slack chat.postMessage
  -> #hermes-bridge channel
  -> @Hermes hermes-bridge + JSON envelope
  -> slack-bot-bridge plugin 驗證 / 去重 / rewrite
  -> Hermes 正常 dispatch
```

## 最短安裝流程

先照 [`docs/INSTALL.md`](docs/INSTALL.md) 拿到這些 Slack ID：

| 你要拿的值 | 長相 | 用在哪裡 |
| --- | --- | --- |
| Bridge channel ID | `C...` | `HERMES_SLACK_BRIDGE_CHANNEL` |
| Hermes bot user ID | `U...` | Slack 訊息裡 `<@U...>` mention Hermes |
| Relay bot user ID | `U...` | `SLACK_ALLOWED_USERS`，讓 Hermes authorization 放行這個 bot |
| Relay bot ID | `B...` | `HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS` |
| Relay app ID | `A...` | `HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS` |
| Workspace / team ID | `T...` | `HERMES_SLACK_BRIDGE_ALLOWED_TEAMS` |

安裝 plugin：

```bash
HERMES_SLACK_BRIDGE_CHANNEL=C0123456789 \
  hermes plugins install pingchesu/hermes-slack-bot-bridge --enable
```

更新 Hermes config，讓 Slack adapter 接受「有 mention Hermes 的 bot message」：

```yaml
# ~/.hermes/config.yaml
slack:
  allow_bots: mentions
  strict_mention: true
```

更新 Hermes env：

```bash
# ~/.hermes/.env
# 原本有誰就保留，後面加上 relay bot 的 U... user_id
SLACK_ALLOWED_USERS=U_YOUR_USER,U_RELAY_BOT_USER

HERMES_SLACK_BRIDGE_CHANNEL=C0123456789
HERMES_SLACK_BRIDGE_ALLOWED_BOT_IDS=B0AAAAAAA
HERMES_SLACK_BRIDGE_ALLOWED_APP_IDS=A0BBBBBBB
HERMES_SLACK_BRIDGE_ALLOWED_TEAMS=T0CCCCCCC
HERMES_SLACK_BRIDGE_HMAC_SECRET=replace-with-a-long-random-secret
```

重啟 Hermes gateway：

```bash
hermes gateway restart
```

## Slack 訊息格式

Relay bot 要送這種訊息到 bridge channel：

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

重點：

- `<@U_HERMES_BOT>` 要用 Hermes bot 的 **Slack user ID** (`U...`)，不是 bot ID (`B...`)。
- `hermes-bridge` 是固定 marker。
- JSON 建議放在 fenced code block：<code>```json</code>。
- `request_id` 用來去重；同一個 request 在 dedup TTL 內不會重複觸發。
- 如果設定 `HERMES_SLACK_BRIDGE_HMAC_SECRET`，每個 envelope 都必須帶正確 `signature`。

## 完整文件

- [`docs/INSTALL.md`](docs/INSTALL.md) — 從零開始安裝，包含 Slack bot user ID 取得方式。
- [`after-install.md`](after-install.md) — `hermes plugins install` 後顯示的快速提醒。

## 安全模型

- **Bridge channel allowlist 是必要條件**：沒有 `HERMES_SLACK_BRIDGE_CHANNEL` 時 plugin 不會處理任何訊息。
- **Sender allowlist 建議正式環境必填**：用 relay bot 的 `bot_id` 或 `app_id` 限制來源。
- **HMAC 建議正式環境必開**：避免同一 channel 裡其他成員手刻 JSON 觸發 Hermes。
- **Hermes 原本的 Slack authorization 還會跑**：relay bot 的 `user_id` (`U...`) 必須在 `SLACK_ALLOWED_USERS` 裡，否則 plugin rewrite 後仍會被 Hermes gateway 擋掉。
- **不接受 free-form bot text**：必須有 mention、`hermes-bridge` marker、JSON envelope。
