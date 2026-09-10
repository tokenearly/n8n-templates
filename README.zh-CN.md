[English](README.md) · 简体中文 · [한국어](README.ko.md)

# Tokenearly n8n 模板

可直接导入的 [n8n](https://n8n.io) 工作流：把加密资产交易所的**上新公告**（以及资讯、X 帖子、信号、价格提醒）无需编写代码地送到 Telegram、Discord、Slack 或 Google Sheets。适合已经在使用 n8n 的交易者、社区运营人员和分析师。

分两类：

- **拉取公开接口——不需要注册任何账号。** 用定时触发器轮询一个免费只读的上新 JSON 接口，覆盖 10 家交易所。无需注册、无需 API key、无需 Bearer Token。想最快把上新提醒接进聊天工具，选这个。
- **接收 Tokenearly Webhook 推送。** 用 Webhook 节点接收推送，校验 Bearer Token，按中英韩三语的上新关键词过滤，并映射四个字段（`title`、`content`、`timestamp`、`url`）。

Last updated: 2026-09-10

## 工作流

### 不需要 Tokenearly 账号

| 文件 | 流程 | 所需凭据 |
|---|---|---|
| [listing-alerts-no-signup-to-telegram.json](listing-alerts-no-signup-to-telegram.json) | 定时触发（每 5 分钟）→ 设置 → HTTP Request（公开上新接口）→ 守卫 → Split Out → Remove Duplicates → 交易所过滤 → 关键词过滤 → 组装消息 → 现货/合约分支发 Telegram，可选 Discord 与 Google Sheets | Telegram Bot API（Discord 与 Sheets 可选） |

它读取 `https://tokenearly.com/api/public/listings.json`——一个免鉴权的免费 JSON 接口，包含 10 家交易所的现货与合约上新。建议从这个模板开始。

所有你可能想改的东西都集中在一个 **Your settings** 节点里：监控哪些交易所、只要现货还是只要合约、关键词过滤、回看多少天、消息用哪种语言、以及是否同时发 Discord 或写入表格。*Remove Duplicates* 会记住已处理过的每个 `permalink`，重启也不会重复提醒；守卫节点会在接口无返回时安静结束这一轮。

### 接收 Tokenearly Webhook 推送

| 文件 | 流程 | 所需凭据 |
|---|---|---|
| [crypto-listing-alerts-to-telegram.json](crypto-listing-alerts-to-telegram.json) | Webhook → Check Bearer Token → Is Listing Alert? → Telegram（HTML 消息） | Telegram Bot API |
| [crypto-listing-alerts-to-discord-and-slack.json](crypto-listing-alerts-to-discord-and-slack.json) | Webhook → Check Bearer Token → Is Listing Alert? → Discord webhook + Slack incoming webhook（HTTP Request 节点） | 无（仅需 webhook URL） |
| [new-listing-to-google-sheets.json](new-listing-to-google-sheets.json) | Webhook → Check Bearer Token → Is Listing Alert? → Map Fields → Google Sheets 追加写入 | Google Sheets OAuth2 |

这三个工作流都会在收到 Webhook 后立即向 Tokenearly 返回 `200`（`responseMode: onReceived`），因此后续转发不会触发 Tokenearly 的 10 秒超时。

## 导入步骤——公开接口工作流

1. 在 n8n 中打开 **Workflows → Add workflow → ⋯ → Import from file**，选择 [listing-alerts-no-signup-to-telegram.json](listing-alerts-no-signup-to-telegram.json)。
2. 添加一个 **Telegram** 凭据（在 [@BotFather](https://t.me/BotFather) 申请 bot token）。
3. 打开 **Your settings** 节点，把你的会话、群组或频道 ID 填进 `telegram_chat_id`。
4. 保存并**激活**。到此结束——这个接口不需要任何凭据。

其余都是可选的，全部在同一个 **Your settings** 节点里：

| 字段 | 含义 |
|---|---|
| `exchanges` | 逗号分隔的交易所 id，如 `binance,okx`；留空表示全部 |
| `listing_type` | `spot` 或 `futures`；留空表示两者都要 |
| `keyword` | 只在标题含该词时提醒，三语标题一起匹配；留空表示不过滤 |
| `lookback_days` | 1–30；首次运行想回补历史就调大 |
| `language` | 消息标题用 `en`、`zh` 还是 `ko` |
| `send_to_discord` + `discord_webhook_url` | 设为 `true` 并粘贴 Discord 频道 webhook URL；不需要 n8n 凭据 |
| `log_to_sheet` | 设为 `true`，然后在 *Append to Google Sheets* 上选 Google Sheets 凭据并填表格 ID |

## 导入步骤——Webhook 工作流

1. 在 n8n 中打开 **Workflows → Add workflow → ⋯ → Import from file**，选择其中一个 JSON 文件（n8n 1.x）。
2. 打开 **Check Bearer Token** 节点，把 `REPLACE_WITH_YOUR_TOKENEARLY_BEARER_TOKEN` 替换为一段足够长的随机字符串。稍后需要在 Tokenearly 中填入同一字符串。
3. 配置目标渠道：
   - Telegram：选择（或新建）一个 Telegram Bot 凭据，并在 *Send to Telegram* 节点上设置 **Chat ID**。
   - Discord / Slack：把 Discord 频道 webhook URL 和 Slack incoming webhook URL 分别粘贴到两个 HTTP Request 节点中。
   - Google Sheets：选择一个 Google Sheets 凭据，设置电子表格 ID，并创建一个名为 `Listings` 的工作表，表头行设为 `received_at | exchange | headline | url | sent_at_unix | content`。
4. 保存并**激活**工作流，然后从 *Tokenearly Webhook* 节点复制 **Production URL**（以 `/webhook/tokenearly-alerts` 结尾）。
5. 在 https://tokenearly.com/dashboard 中打开 **通知渠道 → Webhook**，粘贴 Production URL 和同一个 Bearer Token，保存后点击测试按钮。n8n 中应能看到这次执行记录。

Webhook 是 Tokenearly 的付费套餐渠道。

## 字段映射——公开接口

`listings.json` 的 `items[]` 每一项经 *Split Out* 后在 n8n 中形如：

| 字段 | n8n 表达式 | 示例 |
|---|---|---|
| `exchange` | `{{ $json.exchange }}` | `binance` |
| `exchange_name` | `{{ $json.exchange_name }}` | `Binance` |
| `type` | `{{ $json.type }}` | `spot` 或 `futures` |
| `symbols` | `{{ $json.symbols.join(', ') }}` | `ARB, OP` |
| `title` | `{{ $json.title.zh }}` / `.en` / `.ko` | 对应语言的标题 |
| `published_at` | `{{ $json.published_at }}` | `2026-09-10T08:12:00Z` |
| `source_url` | `{{ $json.source_url }}` | 交易所自己的公告页 |
| `permalink` | `{{ $json.permalink }}` | 固定链接，用作去重键 |

同一套接口还有两个端点，同样免鉴权：

| 端点 | 返回内容 |
|---|---|
| `https://tokenearly.com/api/public/exchanges.json` | 监控的交易所清单、各家的采集方式、近 30 天上新数 |
| `https://tokenearly.com/feed/listings.xml` | 同一份上新数据的 RSS 2.0 |

## 字段映射——Webhook 推送

Tokenearly 每条推送发送一个 JSON 请求体（完整结构见 [tokenearly/webhook-examples/docs/payload.md](https://github.com/tokenearly/webhook-examples/blob/main/docs/payload.md)）。在 n8n 中，Webhook 节点会把它暴露在 `$json.body` 下：

| Tokenearly 字段 | n8n 表达式 | 示例 |
|---|---|---|
| `title` | `{{ $json.body.title }}` | `【币安(Binance)】` |
| `content` | `{{ $json.body.content }}` | 多行文本，区块之间以 `— — — — — — — — — —` 分隔 |
| `timestamp` | `{{ $json.body.timestamp }}` | `1788775210`（Unix 秒，发送时间） |
| `url` | `{{ $json.body.url }}` | 原始公告链接，价格提醒为 `""` |
| Bearer Token 请求头 | `{{ $json.headers.authorization }}` | `Bearer …` |

Google Sheets 工作流使用的派生值：

| 列 | 表达式 |
|---|---|
| `exchange` | `{{ ($json.body.title.match(/【([^】]+)】/) \|\| [])[1] \|\| $json.body.title }}` |
| `headline` | `{{ $json.body.content.split('\n')[0].replace(/^\[(ZH\|EN)\] /, '') }}` |
| `received_at` | `{{ $now.toISO() }}` |

## 自定义过滤规则

*Is Listing Alert?* 节点用于筛出上新公告（即上币公告）类推送：只保留标题或正文中包含 `list`, `listing`, `上线`, `上币`, `新增`, `상장` 任一关键词的推送（不区分大小写）。你可以添加自己的关键词（`Launchpool`、`Alpha`、某个资产代码），也可以删除该节点并把 *Check Bearer Token* 直接连到目标节点，从而转发 Tokenearly 发送的每一条推送。

## 常见问题

**Token 正确但 Bearer Token 校验仍然失败。** n8n 会把请求头名称转成小写；表达式读取的是 `$json.headers.authorization`，并与完整字符串 `Bearer <token>` 比较。请确认中间只有一个空格，且末尾没有多余空白。

**Telegram 返回 "can't parse entities"。** 模板已针对 HTML 解析模式转义了 `&`、`<` 和 `>`。如果你修改过消息表达式，请保留 `.replace(...)` 链式调用，或关闭解析模式。

**能否用 Slack 或 Discord 节点代替 HTTP Request？** 可以。选择 HTTP Request 是因为 incoming webhook URL 不需要 n8n 凭据。

## 数据来源与署名

公开接口以 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 发布，包含标题、分类、时间、代币代码与原文链接，不转载公告正文。署名方式：*Data by Tokenearly (https://tokenearly.com)*。面向程序与 AI 代理的说明见 `https://tokenearly.com/llms.txt`。

## 许可证

MIT © Tokenearly

---

Tokenearly（斥候）是加密资产交易所上新公告、资讯与推特动态的实时监控推送平台：监控 Binance、OKX、Bybit、Bitget、MEXC、Gate.io、HTX、KuCoin、Upbit、Bithumb 10 家交易所公告（币安与 Gate.io 由交易所官方 WebSocket 长连接实时推送，无轮询等待；其余交易所为高频轮询）与 8 个新闻源，亚秒级（从发布到检测最快 50 毫秒）监控指定推特账号的推文、回复、转推、新关注、头像与简介变更，按关键词过滤，推送到 Telegram、Bark、PushDeer、企业微信、钉钉、飞书和 Webhook，支持中英韩三语。
