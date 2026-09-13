[English](README.md) · [简体中文](README.zh-CN.md) · 한국어

# Tokenearly n8n 템플릿

바로 가져올 수 있는 [n8n](https://n8n.io) 워크플로입니다. 암호화폐 거래소의 **토큰 상장 알림**(그리고 뉴스, X 게시물, 시그널, 가격 알림)을 코드 작성 없이 Telegram, Discord, Slack 또는 Google Sheets로 전달합니다. 이미 n8n을 운영 중인 트레이더, 커뮤니티 매니저, 애널리스트를 위한 템플릿입니다.

두 가지 방식이 있습니다.

- **공개 API 폴링 — 어떤 계정도 필요 없음.** 스케줄 트리거가 10개 거래소의 신규 상장을 담은 무료 읽기 전용 JSON API를 폴링합니다. 가입도, API 키도, Bearer Token도 필요하지 않습니다. 상장 알림을 가장 빠르게 채팅으로 받는 방법입니다.
- **Tokenearly Webhook 수신.** Webhook 노드가 푸시된 알림을 받아 Bearer Token을 검증하고, 영어·중국어·한국어 상장 키워드로 필터링한 뒤 네 가지 필드(`title`, `content`, `timestamp`, `url`)를 매핑합니다.

Last updated: 2026-09-10

## 워크플로

### Tokenearly 계정이 필요 없음

| 파일 | 흐름 | 필요한 자격 증명 |
|---|---|---|
| [listing-alerts-no-signup-to-telegram.json](listing-alerts-no-signup-to-telegram.json) | 스케줄(5분마다) → 설정 → HTTP Request(공개 상장 API) → 가드 → Split Out → Remove Duplicates → 거래소 필터 → 키워드 필터 → 메시지 구성 → 현물/선물 Telegram 분기, 선택적 Discord와 Google Sheets | Telegram Bot API(Discord와 Sheets는 선택) |

`https://tokenearly.com/api/public/listings.json`을 읽습니다. 10개 거래소의 현물·선물 신규 상장을 담은 무료 인증 불필요 JSON API입니다. 여기서 시작하세요.

바꾸고 싶은 모든 항목은 **Your settings** 노드 한 곳에 모여 있습니다. 어떤 거래소를 볼지, 현물만인지 선물만인지, 키워드 필터, 며칠까지 되돌아볼지, 메시지 언어, 그리고 Discord로도 보낼지 시트에 기록할지까지. *Remove Duplicates*가 이미 처리한 `permalink`를 모두 기억하므로 재시작 후에도 중복 알림이 없고, 가드 노드가 API 응답이 없을 때 조용히 실행을 종료합니다.

### Tokenearly Webhook 수신

| 파일 | 흐름 | 필요한 자격 증명 |
|---|---|---|
| [crypto-listing-alerts-to-telegram.json](crypto-listing-alerts-to-telegram.json) | Webhook → Check Bearer Token → Is Listing Alert? → Telegram(HTML 메시지) | Telegram Bot API |
| [crypto-listing-alerts-to-discord-and-slack.json](crypto-listing-alerts-to-discord-and-slack.json) | Webhook → Check Bearer Token → Is Listing Alert? → Discord webhook + Slack incoming webhook(HTTP Request 노드) | 없음(webhook URL만 필요) |
| [new-listing-to-google-sheets.json](new-listing-to-google-sheets.json) | Webhook → Check Bearer Token → Is Listing Alert? → Map Fields → Google Sheets 행 추가 | Google Sheets OAuth2 |

이 세 워크플로는 모두 Webhook을 수신하는 즉시 Tokenearly에 `200`을 응답하므로(`responseMode: onReceived`), 전달 과정이 Tokenearly의 10초 타임아웃에 걸리는 일이 없습니다.

## 가져오기 절차 — 공개 API 워크플로

1. n8n에서 **Workflows → Add workflow → ⋯ → Import from file**을 열고 [listing-alerts-no-signup-to-telegram.json](listing-alerts-no-signup-to-telegram.json)를 선택합니다.
2. **Telegram** 자격 증명을 추가합니다([@BotFather](https://t.me/BotFather)에서 봇 토큰 발급).
3. **Your settings** 노드를 열어 본인의 채팅·그룹·채널 ID를 `telegram_chat_id`에 넣습니다.
4. 저장하고 **활성화**합니다. 이것으로 끝입니다. 이 API에는 자격 증명이 필요하지 않습니다.

나머지는 모두 선택 사항이며 같은 **Your settings** 노드에 있습니다.

| 필드 | 의미 |
|---|---|
| `exchanges` | 쉼표로 구분한 거래소 id(예: `binance,okx`), 비우면 전체 |
| `listing_type` | `spot` 또는 `futures`, 비우면 둘 다 |
| `keyword` | 제목에 해당 단어가 있을 때만 알림, 세 언어 제목을 함께 매칭, 비우면 필터 없음 |
| `lookback_days` | 1–30, 첫 실행에서 과거 데이터를 받으려면 늘리세요 |
| `language` | 메시지 제목 언어: `en`, `zh`, `ko` |
| `send_to_discord` + `discord_webhook_url` | `true`로 두고 Discord 채널 webhook URL을 붙여 넣기, n8n 자격 증명 불필요 |
| `log_to_sheet` | `true`로 두고 *Append to Google Sheets*에서 Google Sheets 자격 증명과 스프레드시트 ID를 설정 |

## 가져오기 절차 — Webhook 워크플로

1. n8n에서 **Workflows → Add workflow → ⋯ → Import from file**을 열고 JSON 파일 중 하나를 선택합니다(n8n 1.x).
2. **Check Bearer Token** 노드를 열어 `REPLACE_WITH_YOUR_TOKENEARLY_BEARER_TOKEN`을 충분히 긴 임의의 문자열로 바꿉니다. 같은 문자열을 Tokenearly에도 입력하게 됩니다.
3. 대상 채널을 설정합니다.
   - Telegram: Telegram Bot 자격 증명을 선택(또는 생성)하고 *Send to Telegram* 노드에서 **Chat ID**를 설정합니다.
   - Discord / Slack: Discord 채널 webhook URL과 Slack incoming webhook URL을 두 HTTP Request 노드에 각각 붙여 넣습니다.
   - Google Sheets: Google Sheets 자격 증명을 선택하고 스프레드시트 ID를 설정한 뒤, `Listings`라는 이름의 시트를 만들고 헤더 행을 `received_at | exchange | headline | url | sent_at_unix | content`로 입력합니다.
4. 워크플로를 저장하고 **활성화**한 다음, *Tokenearly Webhook* 노드에서 **Production URL**을 복사합니다(`/webhook/tokenearly-alerts`로 끝납니다).
5. https://tokenearly.com/dashboard 에서 **알림 채널 → Webhook**을 열고 Production URL과 같은 Bearer Token을 붙여 넣은 뒤 저장하고 테스트 버튼을 누릅니다. n8n에 실행 기록이 나타나야 합니다.

Webhook은 Tokenearly의 유료 플랜 채널입니다.

## 필드 매핑 — 공개 API

`listings.json`의 `items[]` 각 항목은 *Split Out* 이후 n8n에서 다음과 같습니다.

| 필드 | n8n 표현식 | 예시 |
|---|---|---|
| `exchange` | `{{ $json.exchange }}` | `binance` |
| `exchange_name` | `{{ $json.exchange_name }}` | `Binance` |
| `type` | `{{ $json.type }}` | `spot` 또는 `futures` |
| `symbols` | `{{ $json.symbols.join(', ') }}` | `ARB, OP` |
| `title` | `{{ $json.title.ko }}` / `.en` / `.zh` | 해당 언어의 제목 |
| `published_at` | `{{ $json.published_at }}` | `2026-09-10T08:12:00Z` |
| `source_url` | `{{ $json.source_url }}` | 거래소 자체 공지 페이지 |
| `permalink` | `{{ $json.permalink }}` | 고정 URL, 중복 제거 키로 사용 |

같은 API에 인증 없이 쓸 수 있는 엔드포인트가 두 개 더 있습니다.

| 엔드포인트 | 반환 내용 |
|---|---|
| `https://tokenearly.com/api/public/exchanges.json` | 모니터링 대상 거래소, 각 거래소의 수집 방식, 최근 30일 상장 건수 |
| `https://tokenearly.com/feed/listings.xml` | 동일한 상장 데이터의 RSS 2.0 |

## 필드 매핑 — Webhook 페이로드

Tokenearly는 알림 한 건마다 JSON 본문 하나를 전송합니다(전체 스키마: [tokenearly/webhook-examples/docs/payload.md](https://github.com/tokenearly/webhook-examples/blob/main/docs/payload.md)). n8n에서는 Webhook 노드가 이를 `$json.body` 아래에 노출합니다.

| Tokenearly 필드 | n8n 표현식 | 예시 |
|---|---|---|
| `title` | `{{ $json.body.title }}` | `【币安(Binance)】` |
| `content` | `{{ $json.body.content }}` | 여러 줄 텍스트, 블록은 `— — — — — — — — — —`로 구분 |
| `timestamp` | `{{ $json.body.timestamp }}` | `1788775210`(Unix 초, 발송 시각) |
| `url` | `{{ $json.body.url }}` | 원본 공지 링크, 가격 알림은 `""` |
| Bearer Token 헤더 | `{{ $json.headers.authorization }}` | `Bearer …` |

Google Sheets 워크플로에서 사용하는 파생 값:

| 열 | 표현식 |
|---|---|
| `exchange` | `{{ ($json.body.title.match(/【([^】]+)】/) \|\| [])[1] \|\| $json.body.title }}` |
| `headline` | `{{ $json.body.content.split('\n')[0].replace(/^\[(ZH\|EN)\] /, '') }}` |
| `received_at` | `{{ $now.toISO() }}` |

## 필터 사용자 지정

*Is Listing Alert?* 노드는 제목 또는 본문에 `list`, `listing`, `上线`, `上币`, `新增`, `상장` 중 하나라도 포함된 알림만 통과시킵니다(대소문자 구분 없음). 원하는 키워드(`Launchpool`, `Alpha`, 특정 티커)를 추가하거나, 이 노드를 삭제하고 *Check Bearer Token*을 대상 노드에 바로 연결하면 Tokenearly가 보내는 모든 알림을 전달할 수 있습니다.

## FAQ

**토큰이 맞는데도 Bearer Token 검증이 실패합니다.** n8n은 헤더 이름을 소문자로 변환합니다. 표현식은 `$json.headers.authorization`을 읽어 전체 문자열 `Bearer <token>`과 비교합니다. 공백이 정확히 하나만 있고 끝에 불필요한 공백이 없어야 합니다.

**Telegram이 "can't parse entities"를 반환합니다.** 템플릿은 HTML 파싱 모드에 맞춰 `&`, `<`, `>`를 이스케이프합니다. 메시지 표현식을 수정했다면 `.replace(...)` 체인을 유지하거나 파싱 모드를 꺼야 합니다.

**HTTP Request 대신 Slack이나 Discord 노드를 사용해도 되나요?** 됩니다. incoming webhook URL은 n8n 자격 증명이 필요 없기 때문에 HTTP Request를 선택했을 뿐입니다.

## 데이터 출처 및 저작자 표시

공개 API는 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)으로 배포됩니다. 제목, 분류, 시각, 토큰 심볼, 원문 공지 링크를 포함하며 공지 본문은 재배포하지 않습니다. 표기 방식: *Data by Tokenearly (https://tokenearly.com)*. 프로그램과 AI 에이전트를 위한 안내는 `https://tokenearly.com/llms.txt`에 있습니다.

## 라이선스

MIT © Tokenearly

---

Tokenearly(토큰얼리)는 암호화폐 거래소의 토큰 상장 공지, 뉴스, X(트위터) 활동을 실시간으로 모니터링하고 알림을 보내는 플랫폼입니다. Binance, OKX, Bybit, Bitget, MEXC, Gate.io, HTX, KuCoin, Upbit, Bithumb 10개 거래소 공지(바이낸스와 Gate.io는 거래소 공식 WebSocket 상시 연결로 실시간 수신해 폴링 대기가 없고, 나머지 거래소는 고빈도 폴링)와 8개 뉴스 소스를 모니터링하고, 지정한 X 계정의 게시물·답글·리포스트·새 팔로우·프로필 사진과 소개 변경을 서브초(게시부터 감지까지 최단 50ms)로 추적해 키워드로 필터링한 뒤 Telegram, Bark, PushDeer, WeCom, DingTalk, Feishu, Webhook으로 한국어·영어·중국어 알림을 제공합니다.
