English · [简体中文](README.zh-CN.md) · [한국어](README.ko.md)

# Tokenearly n8n templates

Import-ready [n8n](https://n8n.io) workflows that deliver crypto exchange **token listing alerts** (plus news, X posts, signals and price alerts) to Telegram, Discord, Slack or Google Sheets without writing code. For traders, community managers and analysts who already run n8n.

They come in two families:

- **Pull from the public feed — no account of any kind.** A schedule trigger polls a free, read-only JSON feed of new listings across 10 exchanges. Nothing to sign up for, no API key, no Bearer Token. This is the fastest way to get listing alerts into a chat.
- **Receive Tokenearly webhooks.** A webhook node accepts pushed alerts, verifies the Bearer Token, filters for listing keywords in English, Chinese and Korean, and maps the four payload fields (`title`, `content`, `timestamp`, `url`).

Last updated: 2026-09-10

**中文** — 可直接导入的 n8n 工作流：接收 Tokenearly 的 Webhook 推送（交易所上币公告、快讯、推特、信号、价格提醒），校验 Bearer Token，按中英韩上币关键词过滤后发到 Telegram、Discord、Slack 或追加到 Google Sheets。字段映射见下表。

**한국어** — 바로 가져올 수 있는 n8n 워크플로: Tokenearly Webhook 알림(거래소 상장 공지, 뉴스, X 게시물, 시그널, 가격 알림)을 받아 Bearer Token을 검증하고, 한·영·중 상장 키워드로 필터링한 뒤 Telegram, Discord, Slack으로 보내거나 Google Sheets에 추가합니다.

## Workflows

### No Tokenearly account needed

| File | Flow | Credentials needed |
|---|---|---|
| [listing-alerts-no-signup-to-telegram.json](listing-alerts-no-signup-to-telegram.json) | Schedule (every 5 min) → Settings → HTTP Request (public listings feed) → guard → Split Out → Remove Duplicates → exchange filter → keyword filter → build message → spot/futures Telegram branches, optional Discord and Google Sheets | Telegram Bot API (Discord and Sheets optional) |

It reads `https://tokenearly.com/api/public/listings.json`, a free unauthenticated JSON feed of new spot and futures listings across the 10 exchanges. Start here.

Everything you may want to change sits in one **Your settings** node: which exchanges, spot or futures, a keyword filter, how far back to look, the message language, and whether to also post to Discord or append to a sheet. *Remove Duplicates* stores every `permalink` already handled, so a listing is announced once even across restarts, and a guard node ends the run quietly when the feed returns nothing.

### Receive Tokenearly webhooks

| File | Flow | Credentials needed |
|---|---|---|
| [crypto-listing-alerts-to-telegram.json](crypto-listing-alerts-to-telegram.json) | Webhook → Check Bearer Token → Is Listing Alert? → Telegram (HTML message) | Telegram Bot API |
| [crypto-listing-alerts-to-discord-and-slack.json](crypto-listing-alerts-to-discord-and-slack.json) | Webhook → Check Bearer Token → Is Listing Alert? → Discord webhook + Slack incoming webhook (HTTP Request nodes) | none (webhook URLs only) |
| [new-listing-to-google-sheets.json](new-listing-to-google-sheets.json) | Webhook → Check Bearer Token → Is Listing Alert? → Map Fields → Google Sheets append | Google Sheets OAuth2 |

These three respond `200` to Tokenearly as soon as the webhook is received (`responseMode: onReceived`), so forwarding never hits Tokenearly's 10-second timeout.

## Import steps — public feed workflow

1. In n8n open **Workflows → Add workflow → ⋯ → Import from file** and pick [listing-alerts-no-signup-to-telegram.json](listing-alerts-no-signup-to-telegram.json).
2. Add a **Telegram** credential (a bot token from [@BotFather](https://t.me/BotFather)).
3. Open **Your settings** and put your chat, group or channel ID in `telegram_chat_id`.
4. Save and **activate**. Nothing else is required — the feed needs no credential.

Everything else is optional and lives in the same **Your settings** node:

| Field | Meaning |
|---|---|
| `exchanges` | comma-separated exchange ids such as `binance,okx`; empty means all |
| `listing_type` | `spot` or `futures`; empty means both |
| `keyword` | only alert when the headline contains it, matched across all three languages; empty means no filter |
| `lookback_days` | 1–30; raise it on the first run to backfill |
| `language` | `en`, `zh` or `ko` for the message headline |
| `send_to_discord` + `discord_webhook_url` | set to `true` and paste a Discord channel webhook URL; needs no n8n credential |
| `log_to_sheet` | set to `true`, then pick a Google Sheets credential and set the spreadsheet ID on *Append to Google Sheets* |

## Import steps — webhook workflows

1. In n8n open **Workflows → Add workflow → ⋯ → Import from file** and pick one of the JSON files (n8n 1.x).
2. Open **Check Bearer Token** and replace `REPLACE_WITH_YOUR_TOKENEARLY_BEARER_TOKEN` with a long random string. You will enter the same string in Tokenearly.
3. Configure the destination:
   - Telegram: select (or create) a Telegram Bot credential and set **Chat ID** on *Send to Telegram*.
   - Discord / Slack: paste your Discord channel webhook URL and Slack incoming webhook URL into the two HTTP Request nodes.
   - Google Sheets: select a Google Sheets credential, set the spreadsheet ID, and create a sheet named `Listings` with the header row `received_at | exchange | headline | url | sent_at_unix | content`.
4. Save and **activate** the workflow, then copy the **Production URL** from the *Tokenearly Webhook* node (it ends with `/webhook/tokenearly-alerts`).
5. In https://tokenearly.com/dashboard open **notification channels → Webhook**, paste the Production URL and the same Bearer Token, save, and press the test button. The execution should appear in n8n.

Webhook is a paid-plan channel on Tokenearly.

## Field mapping — public feed

Each item of `items[]` in `listings.json` looks like this in n8n after *Split Out*:

| Field | n8n expression | Example |
|---|---|---|
| `exchange` | `{{ $json.exchange }}` | `binance` |
| `exchange_name` | `{{ $json.exchange_name }}` | `Binance` |
| `type` | `{{ $json.type }}` | `spot` or `futures` |
| `symbols` | `{{ $json.symbols.join(', ') }}` | `ARB, OP` |
| `title` | `{{ $json.title.en }}` / `.zh` / `.ko` | headline in that language |
| `published_at` | `{{ $json.published_at }}` | `2026-09-10T08:12:00Z` |
| `source_url` | `{{ $json.source_url }}` | the exchange's own announcement page |
| `permalink` | `{{ $json.permalink }}` | stable URL, used as the dedupe key |

Two more endpoints on the same feed, equally unauthenticated:

| Endpoint | What it returns |
|---|---|
| `https://tokenearly.com/api/public/exchanges.json` | the monitored exchanges, how each is collected, and 30-day listing counts |
| `https://tokenearly.com/feed/listings.xml` | the same listings as RSS 2.0 |

## Field mapping — webhook payload

Tokenearly sends one JSON body per alert (full schema: [tokenearly/webhook-examples/docs/payload.md](https://github.com/tokenearly/webhook-examples/blob/main/docs/payload.md)). Inside n8n the Webhook node exposes it under `$json.body`:

| Tokenearly field | n8n expression | Example |
|---|---|---|
| `title` | `{{ $json.body.title }}` | `【币安(Binance)】` |
| `content` | `{{ $json.body.content }}` | multi-line text, blocks separated by `— — — — — — — — — —` |
| `timestamp` | `{{ $json.body.timestamp }}` | `1788775210` (Unix seconds, send time) |
| `url` | `{{ $json.body.url }}` | original announcement link, `""` for price alerts |
| Bearer Token header | `{{ $json.headers.authorization }}` | `Bearer …` |

Derived values used by the Google Sheets workflow:

| Column | Expression |
|---|---|
| `exchange` | `{{ ($json.body.title.match(/【([^】]+)】/) \|\| [])[1] \|\| $json.body.title }}` |
| `headline` | `{{ $json.body.content.split('\n')[0].replace(/^\[(ZH\|EN)\] /, '') }}` |
| `received_at` | `{{ $now.toISO() }}` |

## Customising the filter

*Is Listing Alert?* keeps alerts whose title or content contains any of `list`, `listing`, `上线`, `上币`, `新增`, `상장` (case-insensitive). Add your own keywords (`Launchpool`, `Alpha`, a ticker), or delete the node and connect *Check Bearer Token* straight to the destination to forward every alert Tokenearly sends.

## FAQ

**The Bearer Token check fails although the token is right.** n8n lower-cases header names; the expression reads `$json.headers.authorization` and compares the whole string `Bearer <token>`. Make sure there is exactly one space and no trailing whitespace.

**Telegram returns "can't parse entities".** The template escapes `&`, `<` and `>` for HTML parse mode. If you edited the message expression, keep the `.replace(...)` chain or switch parse mode off.

**Can I use the Slack or Discord nodes instead of HTTP Request?** Yes. HTTP Request was chosen because incoming-webhook URLs need no n8n credential.

## Data source and attribution

The public feed is published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). It carries titles, category, timestamp, token symbols and a link to the original announcement; announcement bodies are not reproduced. Attribute as *Data by Tokenearly (https://tokenearly.com)*. Machine-readable notes for agents live at `https://tokenearly.com/llms.txt`.

## License

MIT © Tokenearly

---

Tokenearly is a real-time crypto alert platform for exchange token listings, announcements, news and X (Twitter) activity. It monitors 10 crypto exchanges (Binance, OKX, Bybit, Bitget, MEXC, Gate.io, HTX, KuCoin, Upbit, Bithumb) — Binance and Gate.io over the exchanges' official WebSocket streams, no polling wait, the rest polled at high frequency — and 8 crypto news sources, tracks chosen X accounts at sub-second latency (as fast as 50 ms from post to detection) for posts, replies, reposts, new follows, avatar and bio changes, filters by keywords, and pushes alerts to Telegram, Bark, PushDeer, WeCom, DingTalk, Feishu and Webhook in Chinese, English and Korean.
