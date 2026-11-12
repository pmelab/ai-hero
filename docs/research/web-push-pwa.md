# Research: Web push via PWA

Issue: pmelab/ai-hero#3. Date: 2026-11-12.

Question: what does push to an installable PWA deliver on iOS, Android and desktop today (install, permission, payload, delivery when closed), and what does it take to serve it from this React Router 8 node app?

Context: Cadence has no native app. PWA push is the mobile push Channel.

## Summary

- One standard stack works everywhere: Push API + service worker + VAPID (RFC 8030/8291/8292). One server code path for all browsers.
- **iOS/iPadOS is the constraint.** Push works only for a Home Screen web app (16.4+), never in a Safari tab. The user must install first, then tap a button to grant permission. Since iOS 26, every site added to the Home Screen opens as a web app, so a manifest is no longer strictly needed (we should still ship one).
- **Android and desktop**: no install needed. Push works from a normal browser tab once the user grants permission.
- **Permission**: ask from a user gesture (click). Firefox and Safari require it. Every push must show a visible notification (`userVisibleOnly: true`), or browsers can revoke the subscription.
- **Payload**: plan for max ~3993 bytes of plaintext JSON (4096-byte encrypted body). Send IDs + title/body + URL, not content.
- **App closed**: Android and iOS wake the app/service worker through the OS push service. Desktop delivers only while the browser process runs. No browser gives a delivery guarantee: push is best effort. Keep an in-app inbox as the source of truth.
- **Server work**: `web-push` npm lib, one VAPID key pair in env, a `push_subscription` table, a subscribe/unsubscribe action, `public/sw.js` + `public/manifest.webmanifest`, delete subscriptions on 404/410.

## Platform matrix

| | iOS / iPadOS | Android (Chrome, Firefox, Samsung) | Desktop (Chrome, Edge, Firefox, Safari) |
|---|---|---|---|
| Min version | 16.4 [1] | current | Safari 16 on macOS 13+ [3]; Chromium/Firefox current |
| Install needed | Yes: Home Screen web app [1] | No [6] | No [6][3] |
| Permission | User gesture required [1][3] | Prompt; gesture strongly advised [7] | Firefox 72+ requires gesture [8]; Safari requires gesture [3] |
| Delivery, app closed | Yes, via Apple push service (`*.push.apple.com`) [1] | Yes, OS wakes browser, then SW [6] | Only while browser process runs [6] |
| Silent push | Not allowed; subscription revoked [3][4] | Chrome rejects `userVisibleOnly:false` [5] | Same; Firefox has quota for non-visible pushes [9] |
| Badge | Badging API works in push handler [1] | — | — |

## Install requirements

**iOS/iPadOS**
- Push needs a Home Screen web app, added via Share → "Add to Home Screen". Since 16.4 third-party browsers with the browser entitlement can offer this too. Not on Shared iPads. [1]
- Before iOS 26: manifest `display` must be `standalone` or `fullscreen`, else the icon opens as a browser bookmark. [1]
- iOS 26: "By default, every website added to the Home Screen opens as a web app"; user can turn "Open as Web App" off. [2]
- Focus settings sync across devices by manifest `id`, so set a stable `id`. [1]
- There is no install prompt API on iOS. We must show our own instructions ("Share → Add to Home Screen").

**Chrome (Android/desktop) installability** — only needed for the install prompt, not for push [10]:
- HTTPS, manifest with `name`/`short_name`, 192px + 512px icons, `start_url`, `display` in fullscreen/standalone/minimal-ui/window-controls-overlay, `prefer_related_applications` not true.
- User clicked the page once and spent 30 s on it. Then `beforeinstallprompt` fires.

## Permission flow

1. Register service worker on load.
2. Show our own "Enable notifications" UI. On click: `Notification.requestPermission()` then `registration.pushManager.subscribe({ userVisibleOnly: true, applicationServerKey: VAPID_PUBLIC })` [5][7].
3. POST the `PushSubscription` JSON (endpoint + `p256dh` + `auth` keys) to the server. The endpoint is a capability URL: keep it secret, protect the POST against CSRF [9].

Rules:
- Gesture: Firefox 72+ shows only a URL-bar icon without one [8]; Safari/iOS require it [1][3]; MDN advises it everywhere [7].
- "Block" is final. The site cannot re-prompt; the user must change settings [5]. So pre-ask in our own UI first, and only call the native prompt after the user says yes.
- Chrome and Edge need `applicationServerKey` and `userVisibleOnly: true` [7].
- Chrome auto-revokes notification permission for low-engagement, high-volume sites (since Oct 2025, Android + desktop). **Installed web apps are excluded.** [11]
- Handle `pushsubscriptionchange` in the SW: subscriptions can expire or be invalidated [9].

## Payload limits

- Push services MUST accept bodies of 4096 bytes; they MAY return 413 above that [12].
- Encryption is `aes128gcm` only; 86-byte header + 1 padding byte + 16-byte tag leaves **max 3993 bytes plaintext** [13].
- Design: small JSON `{ title, body, url, tag }`. Fetch details in the SW or on click if needed.
- Safari Declarative Web Push (iOS/iPadOS 18.4, macOS 15.5): JSON with `"web_push": 8030`, `notification.title`, `navigate` URL, optional `app_badge`. Shows without a SW, and other browsers handle the same JSON in the SW. Declarative messages avoid silent-push penalties. [4] Worth using as our payload shape.

## Delivery reliability when app is closed

- Android: OS wakes the browser, which wakes the SW, whether the app is open or not [6].
- iOS: delivered through Apple's push service to the Home Screen app [1]. Apple publishes no delivery guarantee or rate limit (not found in primary sources).
- Desktop: delivered while the browser runs, even with no window open. Not delivered if the browser is fully quit; messages queue up to TTL [6][12].
- Headers [12][14]:
  - `TTL` (required, seconds): how long the push service queues. `0` = deliver now or drop. Service may shorten it.
  - `Urgency`: `very-low | low | normal | high` (default normal). Lets the device save battery.
  - `Topic` (≤32 chars): a newer message replaces a queued one with the same topic. Good for "N new replies" collapse.
- Responses [12][14]: 201 accepted; 404/410 subscription gone → delete it; 413 too big; 429 rate-limited → honour `Retry-After`.
- Browsers can drop or revoke: silent pushes revoke the subscription (Safari) [3][4]; Firefox gives a quota to non-visible pushes that resets on visit [9]; Chrome engagement auto-revoke for non-installed sites [11].
- Conclusion: push is a best-effort wake-up. The in-app notification list must stay the source of truth.

## What it takes in this app (React Router 8.3, `react-router-serve`, Vite 7, SQLite/Drizzle)

Nothing PWA-related exists yet (no manifest, no SW, `public/` has only favicon + images).

1. **VAPID keys**: generate once (`npx web-push generate-vapid-keys`), store `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` in env. JWT is ES256, `exp` ≤ 24 h, `sub` = `mailto:` or `https:` [14]. Do not use a localhost subject: Apple returns `403 BadJwtToken` [15].
2. **Server lib**: `web-push` (web-push-libs). 4.0.0 released 2026-11-10 (ESM-only, class syntax); 3.6.7 is the long-stable line [15]. Pin 3.6.7 now, or 4.0.0 after it ages. It handles VAPID JWT + aes128gcm encryption.
3. **Storage**: `push_subscription` table (user id, endpoint unique, p256dh, auth, user agent, created, last success). Delete rows on 404/410.
4. **Routes**: a resource route action to save/delete a subscription (CSRF-protected, needs session user). Expose the VAPID public key to the client through the root loader.
5. **Static files**: `public/sw.js` (served at `/sw.js` → root scope) and `public/manifest.webmanifest` with `id`, `name`, `start_url`, `display: "standalone"`, 192/512 icons. Link manifest in `app/root.tsx` `links`. A hand-written SW is enough: `push` → `showNotification` inside `event.waitUntil`, `notificationclick` → `clients.openWindow(url)`, `pushsubscriptionchange` → re-subscribe and POST.
6. **Optional**: `vite-plugin-pwa` 2.0.0 (supports Vite 8 and earlier; Workbox 7) if we also want offline caching. Not needed for push alone.
7. **Endpoint hosts** (if we allowlist outbound): `*.push.apple.com` [1][3], `fcm.googleapis.com` (Chrome, FCM [6]). Firefox and Edge use their own push services (hosts not verified here).
8. **Testing**: localhost is a secure context for desktop. iOS needs a real HTTPS URL (tunnel) and a Home Screen install.

## Open questions

- Is an iOS user expected to install the PWA? If install rate is low, push reaches few iOS users; email fallback likely needed.
- Do we want Declarative Web Push payload shape from day one? (Recommended: yes, no cost elsewhere.)

## Sources

1. WebKit, "Web Push for Web Apps on iOS and iPadOS" (2023) — https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
2. WebKit, "News from WWDC25: Safari 26 beta" — https://webkit.org/blog/16993/news-from-wwdc25-web-technology-coming-this-fall-in-safari-26-beta/
3. WebKit, "Meet Web Push" (2022) — https://webkit.org/blog/12945/meet-web-push/
4. WebKit, "Meet Declarative Web Push" — https://webkit.org/blog/16535/meet-declarative-web-push/
5. web.dev, "Subscribing a user" — https://web.dev/articles/push-notifications-subscribing-a-user
6. web.dev, "Push notifications FAQ" — https://web.dev/articles/push-notifications-faq
7. MDN, `PushManager.subscribe()` — https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe ; `Notification.requestPermission()` — https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static
8. Mozilla Hacks, "Upcoming notification permission changes in Firefox 72" — https://hacks.mozilla.org/2019/11/upcoming-notification-permission-changes-in-firefox-72/
9. MDN, Push API — https://developer.mozilla.org/en-US/docs/Web/API/Push_API
10. web.dev, "Install criteria" — https://web.dev/articles/install-criteria
11. Google, "Reducing notification overload…" (2025-10-10) — https://blog.google/chromium/automatic-notification-permission/
12. RFC 8030, Generic Event Delivery Using HTTP Push — https://www.rfc-editor.org/rfc/rfc8030.html
13. RFC 8291, Message Encryption for Web Push — https://www.rfc-editor.org/rfc/rfc8291.html
14. web.dev, "The Web Push Protocol" — https://web.dev/articles/push-notifications-web-push-protocol
15. web-push-libs/web-push README and v4.0.0 release — https://github.com/web-push-libs/web-push
