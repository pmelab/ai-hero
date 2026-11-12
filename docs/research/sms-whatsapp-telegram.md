# Research: SMS, WhatsApp and Telegram delivery

Issue: pmelab/ai-hero#4. Date: 2026-11-12.

Question: for SMS, WhatsApp and Telegram as notification Channels, which providers fit, what a message costs per country, what WhatsApp Business rules apply (templates, categories, 24h window, opt-in, 2026 pricing), how Telegram account linking works, and which per-Channel constraints a spec must respect?

Context: Cadence sends Notifications to Students, Instructors, Team Admins and Platform Admins. Each user has a country on the profile.

## Summary

- **Telegram is the cheapest and simplest Channel.** The Bot API is free [T2]. The bot cannot message a user first [T1], so the user must link the account: click a `t.me/<bot>?start=<token>` deep link, press Start, and the bot gets `/start <token>` [T3]. No phone number, no sender registration, no templates.
- **WhatsApp: use the Meta Cloud API direct, or Twilio if we want one vendor for SMS + WhatsApp.** Since 2025-07-01 Meta charges per delivered message, by template category and recipient country code [W1]. Outside the 24h customer service window, only pre-approved templates can be sent [W2]. All Cadence notifications are business-initiated, so in practice **every WhatsApp Notification is a template message**, and they are almost all **utility** category.
- **2026 changes that matter**: since 2026-10-01, utility templates sent inside an open customer service window are **charged** (they were free from 2025-07-01) [W1]. Rates change only on the 1st day of a quarter [W1].
- **SMS is the most expensive and the most regulated Channel.** Costs range from ~$0.012/segment (US, incl. carrier fee) to ~$0.11/segment (Germany) at Twilio list price [S1–S4]. The US needs A2P 10DLC brand + campaign registration (or toll-free verification) before any traffic [S5][S6]. India needs DLT registration for domestic routes [S7]. Brazil bans marketing SMS outside 09:00–22:00 and on Sundays [S8].
- **Recommended provider split**: Twilio for SMS (one API for 10DLC, toll-free, alphanumeric senders, global coverage) and either Twilio or Meta Cloud API direct for WhatsApp. Twilio adds $0.005/WhatsApp message on top of Meta fees [P1]. Telegram needs no provider: call the Bot API directly.
- **Spec must**: store explicit per-Channel opt-in (with timestamp + source), handle opt-out (STOP for SMS, block for WhatsApp/Telegram), keep Notification text short and non-promotional, map each Notification Type to a WhatsApp template name + category, and resolve cost/route by the user's **phone country code**, not the profile country (Meta bills by country calling code [W1]).

## Cost per message (USD, list price)

WhatsApp: Meta rate card, effective 2026-10-01 [W3]. SMS: Twilio list price per segment, outbound, Oct 2026 [S1–S4]. Telegram: free [T2].

| Market | WA marketing | WA utility | WA authentication | SMS (Twilio) | Telegram |
|---|---|---|---|---|---|
| United States ("North America") | 0.0250 | 0.0034 | 0.0034 | 0.0083 + carrier fee 0.0035–0.005 [S1] | 0 |
| Germany | 0.1365 | 0.0550 | 0.0550 | 0.112 [S4] | 0 |
| France | 0.0859 | 0.0300 | 0.0300 | — | 0 |
| Rest of Western Europe | 0.0592 | 0.0171 | 0.0171 | — | 0 |
| United Kingdom | 0.0635 | 0.0220 | 0.0220 | — | 0 |
| India | 0.0118 | 0.0014 | 0.0014 (intl: 0.0304) | 0.0832 [S2] | 0 |
| Brazil | 0.0625 | 0.0068 | 0.0068 | 0.0599 [S3] | 0 |

Notes:

- Twilio adds **$0.005 per WhatsApp message** (inbound or outbound) on top of Meta fees [P1]. Bird lists $0.005 per US WhatsApp marketing message plus Meta fees, and $0.0035 per US SMS long-code segment plus carrier fees [P2][P3].
- Service messages (free-form replies in the window) cost the same as utility, but each business phone number gets 1,000 free service messages per month since 2026-10-01 [W1].
- Utility and authentication have volume tiers per market, counted per business portfolio per month [W1].
- SMS cost is per **segment** (160 GSM-7 chars, 70 UCS-2). A message with an accented name or emoji can be 2–3x the price. Twilio also charges $0.001 per failed message [S2][S3].
- Germany is the outlier: WhatsApp utility ($0.055) and SMS ($0.112) are both expensive there. India and Brazil are cheap on WhatsApp and expensive on SMS.

## Providers

| Provider | SMS | WhatsApp | Fit for Cadence |
|---|---|---|---|
| **Meta Cloud API (direct)** | — | Yes; pay Meta rates only [W1] | Cheapest WhatsApp. We own template management, webhooks, business verification. |
| **Twilio** | Yes; US 10DLC/toll-free/short code, alphanumeric senders abroad, India DLT routes [S5–S8] | Yes; Meta fee + $0.005/msg [P1] | One vendor + one API for both. Good docs per country. Best default. |
| **Vonage** | Yes | Yes; one Messages API for SMS, MMS, WhatsApp, Viber, Messenger [P4] | Equivalent to Twilio. Pricing page blocked automated access; not compared. |
| **Bird (MessageBird)** | Yes; US $0.0035/segment + carrier fees [P3] | Yes; $0.005 + Meta fees (US marketing) [P2] | Lower SMS base price in US. Other country rates not published in text form. |
| **Telegram Bot API** | — | — | Direct HTTPS calls, no provider needed [T2]. |

## WhatsApp Business rules

**Pricing model (2026).** Per-message since 2025-07-01. Conversation-based pricing is deprecated [W1]. One charge per delivered message, by category and recipient country calling code. Users pay nothing [W1].

- Template categories: **marketing**, **utility**, **authentication**. Non-template: **service** (and Meta Business Agent since 2026-07-01, $2.00 / 1M tokens) [W1].
- 2026 timeline [W1]: Jan 1 — rate changes, INR billing for India; Jul 1 — BRL billing for Brazil; **Oct 1 — utility in an open window now charged; service beyond 1,000/month/number charged**; rate updates for 40 market-category pairs.
- Meta changes rates only on Jan 1 / Apr 1 / Jul 1 / Oct 1, with ≥1 month notice for rate cards [W1].

**24h customer service window.** A user message or call opens a 24h window; each new user message resets it [W2]. Inside: free-form (service) messages. Outside: "you can only send pre-approved template messages" [W2]. Cadence Notifications start from the platform, so treat every WhatsApp send as a template send.

**Free entry point window.** Free window of up to 7 days, only for users who arrive via Click-to-WhatsApp ads [W1]. Not relevant to Cadence.

**Template approval** [W4]:

- Templates are reviewed automatically on create and on edit. Only `APPROVED` templates can be sent. Review can take up to 24h.
- Quality rating (High / Medium / Low) based on feedback and read rates. Low quality → template **paused**, then **disabled**.
- Limits: 250 templates per account if the business portfolio is unverified; 6,000 if verified. Max 100 template creations per hour. Templates unused for 12 months are archived.

**Categories** [W5]:

- **Utility**: "typically triggered by a user action or request"; must be "non-promotional" and either "specific to or requested by the user" or "essential or critical to the user". Fits: new lesson published in an enrolled course, refund request decided, Team licence seat assigned.
- **Marketing**: anything promotional, or mixed utility + marketing content. Fits: course sale, "come back" nudges.
- **Authentication**: only for one-time passcodes.
- Meta can re-categorize utility → marketing with 1 day notice (no notice for accounts warned for misuse). The template stays approved but is billed as marketing. Appeals within 60 days [W5].

**Opt-in** [W6]:

- Opt-in is required before any message. It can be general (not WhatsApp-specific), but it must clearly state that the person agrees to receive messages and must name the business.
- Any collection method is allowed (web form, SMS, IVR, paper).
- Give clear opt-out instructions per message category and honour them. Meta recommends separate opt-ins per category. Low-quality senders get rate-limited.

**Messaging limits** [W7]:

- Business-initiated messages are capped per **business portfolio** (all numbers share it): 250 unique recipients per rolling 24h at start, then 2,000 / 10,000 / 100,000 / unlimited.
- To reach 2,000: verify the business, or deliver 2,000 high-quality template messages in 30 days. After that, the limit rises automatically when ≥50% is used within 7 days and quality is high.
- Impact: a course launch to more than 250 Students fails on day one unless the business is verified first.

## Telegram bot account linking

Facts:

- "Bots can't start conversations with users. A user must either add them to a group or send them a message first." [T1]
- Deep link: `https://t.me/<bot>?start=<value>`. The bot receives the message `/start <value>`. Value: up to 64 chars, `A-Z a-z 0-9 _ -` only; base64url for anything else [T3].
- Webhooks: `setWebhook` with a `secret_token` (1–256 chars). Telegram sends it in the `X-Telegram-Bot-Api-Secret-Token` header on each call [T2].
- Rate limits: ~1 msg/s per chat, ≤20 msg/min per group, ~30 msg/s for bulk by default. Paid broadcasts allow up to 1,000 msg/s at 0.1 Stars per message above 30/s (needs ≥100,000 Stars and ≥100,000 MAU) [T4].
- Alternative: the Telegram Login library / OpenID Connect flow returns an `id_token` with the Telegram user ID. The `write` scope (OIDC: `telegram:bot_access`) gives the bot permission to message the user. The legacy iframe widget is archived [T5].

Recommended flow (deep link):

1. User clicks "Connect Telegram" on the Notification settings page.
2. Server creates a one-time, short-lived link token (random, ≤64 chars, base64url), stores `{token, userId, expiresAt}`.
3. UI shows `https://t.me/<bot>?start=<token>` (link + QR code for desktop).
4. User opens Telegram, presses **Start**. Bot webhook receives `/start <token>`.
5. Server checks the token (exists, unused, not expired), stores `telegram_chat_id` on the user, marks the token used, and replies "Connected to Cadence as <name>".
6. On send, a `403 Forbidden` response means the user blocked the bot. Mark the Channel disconnected; do not retry.

## SMS constraints by market

**United States** [S5][S6][S9]:

- Alphanumeric sender IDs are not supported. Use 10DLC (local number), toll-free, or short code.
- **A2P 10DLC**: anyone sending application traffic from a 10-digit number must register a **Brand** (company, EIN) and a **Campaign** (use case, sample messages, opt-in/opt-out/help flow). Unregistered traffic gets extra carrier fees and filtering [S5].
- Daily throughput to T-Mobile: Sole Proprietor 1,000; Low-Volume Standard 2,000; Standard 2,000 → unlimited by Trust Score [S5].
- Fees (Twilio product page): Standard brand $44 one-time, Low-Volume / Sole Prop $4; campaign vetting $15 one-time; $1.50–$10 per campaign per month [S10]. Twilio's support article (updated for fees from 2025-08-01) lists $4.50 / $46 brand fees; that page blocked automated access, so confirm in the Twilio Console.
- **Toll-free**: verification is required before any SMS to the US and Canada. Unverified numbers are `Restricted` [S6]. Simpler than 10DLC for one sender.
- T-Mobile fines for non-compliant traffic: up to $10,000 per content violation [S9].

**India** [S7]:

- Domestic route: company and alphanumeric sender ID must be registered on the operators' **DLT** portal. Twilio provisioning takes 10 business days. Sender ID shows with a 2-letter operator prefix (e.g. `VM-CADENC`).
- Unregistered alphanumeric sender IDs are replaced by a random numeric ID.
- International route: no DLT needed, but the sender is a random short code.
- No shortened URLs.

**Brazil** [S8]:

- Alphanumeric sender: optional pre-registration (TIM, Claro, Vivo), ~10 weeks. Short codes ~2–4 weeks. International long codes not supported.
- Marketing SMS banned outside 09:00–22:00 local time and on Sundays. Brazil has 3 time zones.
- Some carriers drop Unicode and replace accented characters.
- HELP/STOP in Portuguese.

**EU (Germany example)** [S4]:

- Alphanumeric sender IDs supported (one-way, no monthly fee); German mobile numbers $30/month.
- No sender registration noted by Twilio. GDPR consent rules still apply to the opt-in record (not researched here).

## What the spec must respect

1. **Opt-in per Channel**, stored with timestamp, source, and the exact consent text. WhatsApp requires the business name in the consent text [W6]. US 10DLC campaigns must describe the opt-in flow [S5].
2. **Opt-out**: SMS STOP/HELP keywords (local language in Brazil and India) [S7][S8]; Telegram/WhatsApp blocks show up as send errors — disable the Channel for that user.
3. **Phone number verification** (OTP) before enabling SMS or WhatsApp. Store E.164. Route and price by phone country code, since Meta bills by calling code [W1].
4. **WhatsApp template per Notification Type**, with variables only. Keep utility templates free of promotional text, or Meta re-categorizes them to marketing [W5]. Template text changes go through re-review [W4].
5. **Quiet hours** in the user's local time for non-urgent Notifications (required for Brazil marketing SMS [S8]; best practice in India [S7]).
6. **Short text**: SMS 160 GSM-7 chars per segment; Telegram 4,096 chars per message [T2]; link back to Cadence for details.
7. **Cost guard**: SMS to DE/IN/BR is 5–10x the US price. Consider limiting SMS to high-priority Notification Types and preferring Telegram/WhatsApp where the user has linked them.
8. **Throughput**: WhatsApp portfolio limit starts at 250 recipients/24h [W7]; Telegram ~30 msg/s bulk [T4]; US 10DLC daily caps by brand tier [S5]. Fan-out to many Students needs a queue with per-Channel rate limits.
9. **Lead time before launch**: Meta business verification, US 10DLC brand + campaign vetting, India DLT (~10 business days), Brazil alphanumeric (~10 weeks).

## Open questions

- Do we need SMS at all in v1, given Telegram is free and WhatsApp utility is cheap in IN/BR/US?
- Meta direct vs Twilio for WhatsApp: $0.005/msg markup vs. building template management ourselves.
- Exact 10DLC fees (two Twilio pages disagree) — check the Twilio Console at registration time.
- Vonage and Bird country rate tables for DE/IN/BR were not available as text; compare only if Twilio is rejected.

## Sources

- [W1] Meta, WhatsApp Business Platform pricing — https://developers.facebook.com/docs/whatsapp/pricing
- [W2] Meta, Cloud API: send messages (customer service window) — https://developers.facebook.com/docs/whatsapp/cloud-api/guides/send-messages
- [W3] Meta, USD list rates CSV (xlsx), "effective October 1, 2026", linked from [W1] as "USD list rates"
- [W4] Meta, Message template guidelines — https://developers.facebook.com/docs/whatsapp/message-templates/guidelines
- [W5] Meta, Template categorization guidelines — https://developers.facebook.com/docs/whatsapp/updates-to-pricing/new-template-guidelines
- [W6] Meta, Getting opt-in — https://developers.facebook.com/docs/whatsapp/overview/getting-opt-in
- [W7] Meta, Messaging limits — https://developers.facebook.com/docs/whatsapp/messaging-limits
- [T1] Telegram, Bots: an introduction — https://core.telegram.org/bots
- [T2] Telegram, Bot API (changelog: 10.4, 2026-11-09) — https://core.telegram.org/bots/api
- [T3] Telegram, Bot features: deep linking — https://core.telegram.org/bots/features#deep-linking
- [T4] Telegram, Bots FAQ: limits — https://core.telegram.org/bots/faq
- [T5] Telegram, Login — https://core.telegram.org/widgets/login
- [S1] Twilio, SMS pricing US — https://www.twilio.com/en-us/sms/pricing/us
- [S2] Twilio, SMS pricing India — https://www.twilio.com/en-us/sms/pricing/in
- [S3] Twilio, SMS pricing Brazil — https://www.twilio.com/en-us/sms/pricing/br
- [S4] Twilio, SMS pricing Germany — https://www.twilio.com/en-us/sms/pricing/de
- [S5] Twilio, A2P 10DLC — https://www.twilio.com/docs/messaging/compliance/a2p-10dlc
- [S6] Twilio, Toll-free verification — https://www.twilio.com/docs/messaging/compliance/toll-free/console-onboarding
- [S7] Twilio, India SMS guidelines — https://www.twilio.com/en-us/guidelines/in/sms
- [S8] Twilio, Brazil SMS guidelines — https://www.twilio.com/en-us/guidelines/br/sms
- [S9] Twilio, US SMS guidelines — https://www.twilio.com/en-us/guidelines/us/sms
- [S10] Twilio, A2P 10DLC product page (fees) — https://www.twilio.com/en-us/phone-numbers/a2p-10dlc
- [P1] Twilio, WhatsApp pricing — https://www.twilio.com/en-us/whatsapp/pricing
- [P2] Bird, Pricing — https://bird.com/en-us/pricing
- [P3] Bird, SMS pricing — https://bird.com/pricing/sms.md
- [P4] Vonage, SMS API overview — https://developer.vonage.com/en/messaging/sms/overview
