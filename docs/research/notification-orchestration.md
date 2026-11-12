# Notification orchestration: build vs buy

Research for [#5](https://github.com/pmelab/ai-hero/issues/5), part of map [#1](https://github.com/pmelab/ai-hero/issues/1). Checked 2026-11-12 against vendor pricing pages, vendor docs, and GitHub repos. Prices change; re-check before you sign anything.

Our Channels (see `GLOSSARY.md`): in-app inbox with a real-time feed, email, web push to a PWA, SMS, WhatsApp, Telegram.

## TL;DR

- **No vendor covers all six Channels except Novu.** Novu is also the only one with a Telegram provider and the only one with an open-source (MIT) core you can self-host.
- Knock, Courier, SuprSend: strong inbox + preferences, but **no Telegram**. You build Telegram yourself next to them.
- OneSignal is a push/email/SMS sender, **not an orchestration platform for a web inbox**. No WhatsApp, no Telegram, no web inbox listed.
- At small scale (≤10k sends/month) Novu, Knock, Courier, and SuprSend are free. The first real price step is Novu Pro ($30/mo) vs. Knock Starter / SuprSend Business / MagicBell Startup (about $250/mo) vs. Courier pay-as-you-go ($0.005/send).
- **Channel provider costs are separate in every option** (Postmark, Twilio, Meta WhatsApp). The platform only orchestrates.
- Build-it-yourself is a medium-size project: a notifications table, a job queue, a preference model, a real-time feed, and four new provider adapters. Web push and Telegram are cheap to add. WhatsApp and SMS carry the most compliance work, and you carry that work with a vendor too.

## Channel coverage

| Channel | Novu | Knock | Courier | SuprSend | OneSignal | MagicBell |
| --- | --- | --- | --- | --- | --- | --- |
| In-app inbox, real-time | Yes, `<Inbox />` | Yes, feed | Yes, Courier Inbox | Yes, Inbox | Not listed for web | Yes |
| Email via Postmark | Yes | Yes | Yes | Yes | Own email product | Email listed |
| Web push (PWA) | FCM, OneSignal, Pushpad | FCM (web supported) | FCM, OneSignal ("web" in push) | Web push channel | Yes (core product) | Yes |
| SMS | ~40 providers incl. Twilio | Yes, incl. Twilio | Twilio, Vonage, Plivo, Sinch, ... | Yes | OneSignal SMS or Twilio | Yes |
| WhatsApp | WhatsApp Business (Meta Cloud API) | WhatsApp (Meta), or via Twilio | WhatsApp via Twilio | Yes (Karix, Gupshup, ...) | Not mentioned | Not mentioned |
| Telegram | **Yes** | No | No | No | Not mentioned | Not mentioned |

Sources:
- Novu provider enums, `next` branch: [chat.ts](https://github.com/novuhq/novu/blob/next/packages/shared/src/consts/providers/channels/chat.ts) (includes `Telegram`, `WhatsAppBusiness`), [push.ts](https://github.com/novuhq/novu/blob/next/packages/shared/src/consts/providers/channels/push.ts) (`FCM`, `OneSignal`, `Pushpad`, `APNS`, `EXPO`, ...), [email.ts](https://github.com/novuhq/novu/blob/next/packages/shared/src/consts/providers/channels/email.ts) (`Postmark`), [sms.ts](https://github.com/novuhq/novu/blob/next/packages/shared/src/consts/providers/channels/sms.ts).
- Novu [Telegram docs](https://docs.novu.co/platform/integrations/chat/telegram): one bot per integration; the subscriber must open the bot and tap **Start** (via `TelegramConnectButton` or a link endpoint) before delivery works. [WhatsApp docs](https://docs.novu.co/platform/integrations/chat/whats-app): Meta access token + phone number ID; templates are approved in Meta Business Manager.
- Knock [integrations overview](https://docs.knock.app/integrations/overview): chat = Slack, Teams, Discord, plus [WhatsApp](https://docs.knock.app/integrations/chat/whatsapp) ("Sent" status only, no delivery tracking); push = APNs, FCM. [FCM docs](https://docs.knock.app/integrations/push/firebase) say FCM sends "mobile or web push". [Knock llms.txt](https://docs.knock.app/llms.txt) also lists "WhatsApp via Twilio" and "Send web push with FCM" tutorials. No Telegram anywhere.
- Courier [integrations overview](https://www.courier.com/docs/external-integrations/integrations-overview) and [chat providers](https://www.courier.com/docs/integrations/direct-message/overview.md): Slack, Teams, Discord, WhatsApp (through Twilio), Messenger, Viber, Stream Chat. No Telegram.
- SuprSend [docs index](https://docs.suprsend.com/_llms/documentation.txt): email (incl. [Postmark](https://docs.suprsend.com/docs/postmark.md)), SMS, WhatsApp, mobile + web push, Inbox, Slack, Teams. No Telegram.
- OneSignal [pricing](https://onesignal.com/pricing) and [SMS docs](https://documentation.onesignal.com/docs/en/sms-messaging): push, email, SMS/RCS, mobile in-app. No WhatsApp or Telegram mentioned.
- MagicBell [pricing](https://www.magicbell.com/pricing): Inbox, web push, mobile push, Slack, SMS, email.

## User preference management

- **Novu**: `<Preferences />` component; subscribers choose per workflow and per channel. Included in self-hosted community edition too. ([Inbox docs](https://docs.novu.co/platform/inbox/overview), [self-hosted vs cloud](https://github.com/novuhq/novu/blob/next/docs/community/self-hosted-and-novu-cloud.mdx))
- **Knock**: preferences on all plans; per-tenant preferences on Enterprise. ([pricing](https://knock.app/pricing))
- **Courier**: hosted/embedded preference center (topics x channels). The pricing page does not gate it. ([Inbox overview](https://www.courier.com/docs/platform/inbox/inbox-overview))
- **SuprSend**: full preference API (channel, category, tenant, object levels) and React components, **but only from the Business plan ($250/mo)**. ([pricing](https://www.suprsend.com/pricing), [preference API](https://docs.suprsend.com/llms.txt))
- **OneSignal**: subscription/tag-based. It has no per-category x per-channel preference center like the others.

## Pricing at small / medium scale

The unit differs per vendor. Novu counts **workflow runs** (one trigger, many channels). Knock counts **messages delivered per channel**. Courier counts **sends** (one send = one count, even across channels). SuprSend counts **notifications**. MagicBell counts **deliveries per channel**.

| Vendor | Free | First paid tier | Overage |
| --- | --- | --- | --- |
| Novu | 10k runs/mo, 24h activity retention, Novu branding | Pro from $30/mo, 30k runs, 7d retention; Team from $250/mo, 250k runs, 90d | $1.20 / 1k runs |
| Knock | 10k messages/mo, branding | Starter $250/mo, 50k messages | $0.005 / message |
| Courier | 10k sends/mo | Business: pay-as-you-go, $0.005 / send from the first send | — (Enterprise below $0.005) |
| SuprSend | 10k notifications/mo, branded inbox, **no preferences** | Essentials $100/mo, 50k (no preferences, no digest); Business $250/mo, 50k (preferences, digest) | $2 / 1k (Essentials), $5 / 1k (Business) |
| OneSignal | 10k email sends, web push up to 10k subscribers per send | Growth from $19/mo; web push $0.004 / subscriber; email $1.50 / 1k after 20k | SMS price via sales |
| MagicBell | 1k deliveries/mo | Startup $249/mo, 50k deliveries | $0.0025 / delivery |

Sources: [Novu pricing](https://novu.co/pricing), [Knock pricing](https://knock.app/pricing), [Courier pricing](https://www.courier.com/pricing), [SuprSend pricing](https://www.suprsend.com/pricing), [OneSignal pricing](https://onesignal.com/pricing), [MagicBell pricing](https://www.magicbell.com/pricing).

Worked example, 100k Notifications/month, each fanning out to about 2 Channels (in-app + one more):
- Novu Pro: about $30 + 70k x $1.20/1k ≈ **$114/mo** (counts runs, not channels).
- Courier Business: 100k x $0.005 ≈ **$500/mo**.
- Knock Starter: about 200k messages → $250 + 150k x $0.005 ≈ **$1,000/mo**.
- SuprSend Business: depends on whether each channel counts. The pricing page does not say. **$250–$1,000/mo**.
- Plus provider costs in every case, also for build (next section).

### Channel provider costs (pay these with or without a platform)

- **Postmark**: Basic $15/mo for 10k emails, then $1.80 / 1k. Broadcast emails must use a separate message stream from transactional. ([pricing](https://postmarkapp.com/pricing))
- **Twilio SMS (US)**: $0.0083 per segment plus carrier fees of $0.0035–$0.005. US A2P 10DLC registration fees apply. ([pricing](https://www.twilio.com/en-us/sms/pricing/us))
- **WhatsApp (Meta)**: per delivered message since 2025-07-01, priced by category (marketing / utility / authentication) and country. Business-initiated messages must be templates. Free-form messages are allowed only inside the 24h customer service window. **Since 2026-10-01 Meta charges again for in-window utility messages, and for service messages beyond 1,000/month per number.** ([Meta pricing](https://developers.facebook.com/docs/whatsapp/pricing))
- **Web push**: free (browser push services). **Telegram Bot API**: free.

## Self-hosting / open source

- **Novu**: open core. Core is MIT (`LICENSE-MIT`); `/enterprise` uses a commercial license (`LICENSE-ENTERPRISE`). Repo: [novuhq/novu](https://github.com/novuhq/novu), about 40k stars, active (pushed 2026-11-12). The community self-hosted edition includes Inbox, Bell, Preferences component, digest, all channels, code-based workflows. **It lacks:** removing Novu branding, snooze, email activity tracking, translations, more than 2 environments, SSO/MFA/RBAC, and it allows **1 team member**. ([comparison doc](https://github.com/novuhq/novu/blob/next/docs/community/self-hosted-and-novu-cloud.mdx)) Infra: API, Worker, and WS services + dashboard, MongoDB, 2 Redis clusters (one with AOF), and S3-compatible storage; single-VM minimum 4 vCPU / 8 GB. ([self-hosting overview](https://docs.novu.co/community/self-hosting-novu/overview)) Novu's pricing page lists "self-hosted deployment" under Enterprise, which means a supported, licensed self-host.
  - **Fit for us**: Mongo + 2x Redis + S3 is a heavy add-on for a single-SQLite app ([ADR 0006](../adr/0006-one-sqlite-file-through-drizzle.md)).
- **Knock**: hosted only. Pricing page mentions no self-host.
- **Courier**: hosted only. No self-host on the pricing page.
- **SuprSend**: self-hosted and BYOC (beta) **only on Enterprise**.
- **OneSignal, MagicBell**: hosted only.

## React components for an inbox

| Vendor | Package (npm latest, 2026-11-12) | Notes |
| --- | --- | --- |
| Novu | `@novu/react` 3.19.2 | `<Inbox />`, `Bell`, `Notifications`, `Preferences`; hooks for headless; React 18 and 19. `@novu/js` for non-React. ([docs](https://docs.novu.co/platform/inbox/overview)) |
| Knock | `@knocklabs/react` 0.15.1 | Real-time feed component over sockets; `useNotifications` hook; `@knocklabs/client` for headless. ([docs](https://docs.knock.app/in-app-ui/react/feed)) |
| Courier | `@trycourier/courier-react` 9.3.5 | `CourierInbox`, `useCourier`; real-time read/archive sync across devices; older `@trycourier/react-inbox` is deprecated. ([docs](https://www.courier.com/docs/platform/inbox/inbox-overview)) |
| SuprSend | `@suprsend/react` 1.5.0 | Drop-in and headless inbox, JWT auth; replaces `@suprsend/react-inbox`. ([docs](https://docs.suprsend.com/docs/react-inbox-integration.md)) |
| OneSignal | `react-onesignal` 3.5.6 | Push/permission SDK. Not an inbox. |

Our app runs React 19.2 (`package.json`). Novu says it supports React 19. The others do not state it on the pages checked.

## Lock-in and data export

What you would have to move out: workflow definitions, templates, subscriber/channel data (push tokens, phone numbers, Telegram chat IDs), preferences, and in-app feed history.

- **Novu**: lowest lock-in. Workflows can live in code (Framework / code-based workflows). The MIT core is the exit path: you can self-host the same API. Subscribers and preferences are reachable through the REST API.
- **Knock**: workflows and templates sync to the repo with the Knock CLI (`knock pull` / `knock push`, CI/CD promotion). ([CLI docs](https://docs.knock.app/developer-tools/knock-cli.md)) **Warehouse export and streaming of analytics/observability data are Enterprise only.** ([warehouse export](https://docs.knock.app/integrations/extensions/warehouse-export.md), [pricing](https://knock.app/pricing)) Feed retention is unlimited and configurable on all plans.
- **Courier**: templates and journeys are stored in the workspace and referenced by ID. API can create and edit them. Data-warehouse integrations are Enterprise. ([pricing](https://www.courier.com/pricing), [docs index](https://www.courier.com/docs/llms.txt))
- **SuprSend**: log sync to Segment or a warehouse needs Business. Its docs include a [MagicBell → SuprSend migration guide](https://docs.suprsend.com/docs/migrate-from-magicbell-to-suprsend.md), which shows that a migration means mapping users, templates, workflows, preferences, and the inbox one by one.
- **General**: each vendor stores in-app feed history and preferences in its own schema. To keep the GLOSSARY rule "every Notification appears in-app" as our record, store a copy of each Notification in our own SQLite as well. Then the vendor is only a delivery engine, and you can swap it.

## Build it ourselves

Today: `app/lib/mailer.server.ts` has a `MailTransport` interface (`send(email)`), a Postmark transport (`/email`, `MessageStream: "outbound"`, text body only), and an `.outbox/` file transport for dev and tests. Route actions call `sendEmail` inline, in the request (for example `courses.$slug.refund.ts`, `admin.refunds.tsx`). There is no queue, no retry, and no real-time transport.

What a build needs on top of that:

1. **Notification record (SQLite/drizzle)**: `notifications` (recipient, type, payload, read/archived/seen timestamps) is the in-app inbox and the source of truth. Add `notification_deliveries` (channel, provider message id, status, attempts, error) for delivery tracking.
2. **Dispatch + queue**: a trigger function writes the Notification, then enqueues one delivery per enabled Channel. You need retries with backoff and idempotency keys outside the request. With one SQLite file, a DB-backed job table polled by a worker in the node process is enough. Most vendors give you this part: workflows, delays, digests, and batching. Digests are still an open question on map #1.
3. **Preferences**: a table keyed by (user, notification category, channel), with defaults per category. Some Notifications are mandatory (for example refund decisions). Add a settings UI and unsubscribe links (email `List-Unsubscribe`, SMS STOP, WhatsApp opt-in).
4. **Real-time in-app feed**: SSE or WebSocket from the React Router node server, with an unread count and mark-read actions. See the sibling research `docs/research/realtime-transport.md` (#6).
5. **Channel adapters** behind one interface, like `MailTransport`:
   - **Email**: exists. Add HTML/template rendering, a separate Postmark broadcast stream for Platform Admin broadcasts, and bounce/delivery webhooks.
   - **Web push**: standard Push API + service worker + VAPID. The [`web-push`](https://github.com/web-push-libs/web-push) npm package is at 4.0.0 (MPL-2.0, maintained). Store `PushSubscription` per device and delete it on 404/410. On iOS/iPadOS, web push works only when the app is added to the Home Screen with a manifest `display: standalone` (iOS 16.4+), and permission must come from a user gesture. ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)) See `docs/research/web-push-pwa.md` (#3).
   - **SMS**: Twilio (or similar) REST call plus a status callback webhook. Phone number collection and verification, US 10DLC registration, and STOP handling.
   - **WhatsApp**: Meta Cloud API (token + phone number ID). Each business-initiated message must use a pre-approved template per language. Handle the 24h window rule and per-message pricing. ([Meta](https://developers.facebook.com/docs/whatsapp/pricing))
   - **Telegram**: create a bot with BotFather. Link accounts with a deep link `https://t.me/<bot>?start=<token>` (token max 64 chars, `A-Za-z0-9_-`), and receive `/start <token>` by webhook to store the chat ID. Users must start the bot first. Plan for flood-limit retries. ([Telegram bot features](https://core.telegram.org/bots/features#deep-linking))
6. **Ops**: provider webhooks (delivery, bounce, STOP), logs per delivery, retention cleanup (open on map #1), translations (open on map #1).

**The work is the same with build or buy**: phone verification, consent and opt-in per Channel, WhatsApp templates and Meta business verification, the Telegram bot, the PWA service worker, and every provider contract and its bill. A vendor removes items 2–4 and part of 5 (adapters, retries, preference UI, inbox UI). It does not remove the compliance and provider setup.

## Options to decide between

1. **Buy Novu Cloud** (Pro at $30/mo to start). Only vendor with all six Channels incl. Telegram. Prebuilt React 19 Inbox + Preferences. MIT self-host is the exit path.
2. **Buy Knock / Courier / SuprSend + build Telegram ourselves.** More mature inbox/preferences, but higher price per message and a second delivery path for Telegram.
3. **Self-host Novu community edition.** No platform fee. But you run Mongo + 2x Redis + S3 next to SQLite, it allows 1 dashboard team member, and it has Novu branding in the inbox.
4. **Build on Postmark + `web-push` + Twilio + Meta Cloud API + Telegram Bot API**, with notifications, deliveries, preferences, and a jobs table in SQLite. No vendor fees or lock-in. Most engineering work, and the most code to maintain.

## Gaps not verified

- Knock, Courier, SuprSend: React 19 support not stated on the pages checked.
- SuprSend: whether a multi-channel notification counts once or once per channel.
- OneSignal SMS and WhatsApp prices (sales only).
- Novu Inbox real-time transport: docs do not name it; the self-host infra has a dedicated `WS` service.
