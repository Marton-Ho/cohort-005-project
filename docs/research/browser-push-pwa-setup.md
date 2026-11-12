# Browser push and PWA setup for Cadence

Research for issue #4 (child of map #1, "Notifications for Students and Instructors"). Researched 2026-11-12.
The repo had no existing convention for research notes, so this file starts `docs/research/`.

## Short answer

To receive browser push, Cadence needs five things:

1. A service worker at `/sw.js`.
2. A web app manifest with `display: "standalone"`, plus 192px and 512px icons.
3. One VAPID key pair.
4. A `push_subscriptions` table.
5. A "Turn on notifications" button that calls `Notification.requestPermission()` and then `pushManager.subscribe()`.

The server sends pushes with the `web-push` npm package.

Desktop Chrome, Edge, Firefox and Safari, and Android browsers, can receive push from a normal tab. iOS and iPadOS deliver push only to a site the user has added to the Home Screen. That has been true since iOS 16.4, and nothing changed in iOS 18, 26 or 27.

The repo has none of these pieces today. A rough estimate is 3 to 5 developer-days for working push on every platform, including the iOS install hint. Provider integration (Novu) and the Notification settings UI are extra.

## 1. What each platform needs

| Platform | Push works from | Notes |
| --- | --- | --- |
| Chrome, Edge (desktop and Android) | A normal tab, no install needed | Chromium asks for an `applicationServerKey` (VAPID public key) [web.dev subscribing]. Installing the app is optional; the install criteria are in §3. |
| Firefox (desktop and Android) | A normal tab | Since Firefox 72 the permission prompt is a small icon in the address bar, not a pop-up. It appears "when you interact with the site" [Firefox 72 notes]. |
| Safari on macOS | A normal tab (Safari 16 and later) or a Web App on the Mac | Same W3C Web Push, sent through the Apple Push Notification service (APNs) [WebKit 13878]. |
| iOS and iPadOS 16.4 and later | **The Home Screen web app only** | "adding support for Web Push to Home Screen web apps" [WebKit 13878]. Safari tabs cannot subscribe. |

Rules shared by every platform:

- **HTTPS (secure context).** Push and notifications work only over HTTPS. Plain `localhost` also counts, so the dev server works [MDN requestPermission] [MDN installable].
- **The permission prompt must follow a user action.** WebKit: "as long as that request is in response to direct user interaction" [WebKit 13878]. MDN: "the request should be made in response to user interaction", with the call made inside a click handler [MDN requestPermission]. Never ask on page load.
- **`userVisibleOnly: true` is required.** Every push must show a notification; silent push is not allowed [web.dev subscribing].
- **Feature-detect.** Check for `serviceWorker in navigator` and `PushManager in window`. Do not sniff the browser [web.dev subscribing] [WebKit 13878].
- **Firewall.** Apple's push endpoints are `*.push.apple.com`, and the sending server must be allowed to reach them [WebKit 13878]. Chrome uses FCM endpoints, Firefox uses Mozilla autopush, and Edge uses WNS. The server never chooses the endpoint; it POSTs to whatever endpoint the subscription holds.
- **No Apple Developer Program account is needed** [WebKit 13878].

## 2. The pieces

### 2.1 Service worker

- **Serving it.** Put the file at `public/sw.js`. Vite serves `public/` "at root path `/` during dev" and copies it "to the root of the dist directory as-is", without hashing [Vite assets]. In production, `react-router-serve` serves the client build directory with plain `express.static`, so `/sw.js` comes from `build/client/sw.js` with no long-lived cache [`node_modules/@react-router/serve/dist/cli.js` lines 115–120]. Only `/assets/*` gets `immutable, maxAge: 1y`; `public/` itself is served with `maxAge: 1h`. The file must sit at the site root so that its scope covers `/`.
- **No plugin needed.** `vite-plugin-pwa` (Workbox) is optional. Cadence does not need offline caching for push, and a hand-written worker of about 40 lines is enough. Consider the plugin only if offline lessons ever become a goal.
- **What the worker does:**
  - On `push`: read the JSON payload and call `self.registration.showNotification(title, { body, data: { url } })`.
  - On `notificationclick`: run `clients.openWindow(url)`, or focus an existing tab.
  - On `pushsubscriptionchange`: re-subscribe and send the new subscription to the server.
- **Registering it.** Call `navigator.serviceWorker.register('/sw.js')` from a client-only `useEffect` in `app/root.tsx`. It must not run during SSR.

### 2.2 Web app manifest

- Add `public/manifest.webmanifest` and link it from `links` in `app/root.tsx` with `{ rel: "manifest", href: "/manifest.webmanifest" }`. Every page must reference the manifest [MDN installable].
- For Chromium installability, the manifest needs `name` or `short_name`, `icons` (it "must contain a 192px and a 512px icon"), `start_url`, and `display` or `display_override`. `prefer_related_applications` must be false or absent [MDN installable].
- A service worker is "not a requirement for a PWA to be installable" [MDN installable]. Push needs one anyway.
- **iOS:** a manifest with `display` set to `standalone` or `fullscreen` makes the site a Home Screen web app [WebKit 13878]. Set a stable `id`; iOS uses it to sync Focus settings across devices and to tell installs apart [WebKit 13878] [WebKit 16.4].
- **iOS 26:** "every website added to the Home Screen opens as a web app" by default, and "If you define your icons in the manifest, they're used!" [WebKit WWDC25]. Users can still turn "Open as Web App" off and get a plain bookmark.
- **Icons.** Today the repo has only `public/favicon.svg`. PNG icons at 192 and 512, a maskable variant, and a 180×180 `apple-touch-icon` are needed.

### 2.3 VAPID keys

- Generate them once with `npx web-push generate-vapid-keys`, and use them "for all future messages you send" [web-push README].
- Store `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` and `VAPID_SUBJECT` as environment variables. Send the public key to the client through a loader.
- `VAPID_SUBJECT` must be a `mailto:` or `https:` URI with a real contact. Apple may reject `localhost`-style subjects with `403 BadJwtToken` [web-push README].
- If the keys are rotated, every existing subscription is lost, because each subscription is bound to the public key [web.dev subscribing].

### 2.4 Subscription storage

- The client posts `JSON.stringify(subscription)` to the server. That holds the `endpoint` and `keys.p256dh` / `keys.auth`, which the server needs to encrypt payloads [web.dev subscribing].
- Proposed Drizzle table `push_subscriptions` (the `database-migrations` skill applies):

  | Column | Notes |
  | --- | --- |
  | `id` | Primary key |
  | `userId` | Foreign key to the user |
  | `endpoint` | Unique |
  | `p256dh` | Encryption key |
  | `auth` | Encryption key |
  | `userAgent` / label | For example "iPhone web app" |
  | `createdAt` | |
  | `lastSuccessAt` | |

  A user can have many subscriptions, one per device or browser.
- **Pruning.** When a push service answers 404 ("expired") or 410 ("no longer valid and should be removed"), delete the row [web.dev protocol] [RFC 8030 §7.3].
- **Sending.** Use `webpush.sendNotification(sub, payload, { TTL, urgency, topic })` [web-push README]. The TTL header is mandatory [RFC 8030 §5.2], and the push service must accept payloads of at least 4096 bytes [web.dev protocol].
  - `topic` lets the push service coalesce messages, which suits "at most one Notification a day".
  - Set a TTL of hours, not the four-week default, so a stale Streak Notification never arrives late.
- **Which side owns sending.** If Novu is the provider (map #1 baseline), Novu's push integration may hold the VAPID keys and subscriptions instead. That choice belongs to the provider ticket. The client-side pieces above are needed either way.

### 2.5 Permission UX

- Show a "Turn on notifications" button in Notification settings, or a contextual banner after the first Watched video. Call `Notification.requestPermission()` inside its click handler, then `pushManager.subscribe({ userVisibleOnly: true, applicationServerKey })`.
- Explain the value first: "Most users grant permission if they understand why the app requests it" [web.dev subscribing]. If the user blocks the prompt, the site cannot easily ask again.
- On iOS, when the page runs in a Safari tab, show the install hint (§3) instead of the button, because `PushManager` is not available there.

### 2.6 Declarative Web Push (Apple, optional)

- iOS and iPadOS 18.4 and macOS Safari 18.5 accept a JSON payload that the browser displays without running a service worker [WebKit 16535].
  - The format: `{"web_push": 8030, "notification": {"title": ..., "navigate": url, ...}, "app_badge": n}`.
  - Subscribing works via `window.pushManager`.
  - If a service worker exists, it still receives the push event and may replace the notification [WebKit 16535].
- Recommendation: **send payloads in the declarative format and keep the service worker.** Older browsers and non-Apple browsers use the worker; newer Apple builds display it directly. The two approaches are compatible.

## 3. Install flow on iOS and the install hint

**How a user installs:**

- **iOS 18:** In Safari, tap Share, then Add to Home Screen. Since iOS 16.4, third-party browsers can also offer "Add to Home Screen" in their Share menu [WebKit 16.4].
- **iOS 26:** The Add to Home Screen sheet now has an "Open as Web App" toggle, and it is on by default for every site [WebKit WWDC25].
  - In Safari 26's redesigned toolbar, Share sits behind the "…" (More) button. This comes from press coverage, because Apple's support guide did not render for fetching. Check it on a device.
- **iOS 27** (shipped autumn 2026): the WebKit WWDC26 post lists no changes to web apps or push [WebKit WWDC26].

**Detecting standalone mode:**

- Use `window.matchMedia('(display-mode: standalone)').matches`. This media feature has been Baseline since January 2020 [MDN display-mode].
- Older iOS also exposes the non-standard `navigator.standalone`; check both.

**The install hint:** show it when the device is iOS or iPadOS, the page is not standalone, and `PushManager in window` is false. Show a small dismissible card ("Add Cadence to your Home Screen to get Streak Notifications") with the iOS 18 or iOS 26 steps. Remember the dismissal per device in `localStorage`.

- Chromium also offers `beforeinstallprompt` for a custom install button.
- Safari and Firefox do not support `beforeinstallprompt`, and push does not require install outside iOS.

## 4. EU and DMA status

- In February 2024, Apple announced that iOS 17.4 would remove Home Screen web apps in the EU. On 1 March 2024 it reversed that and kept the existing capability, built on WebKit (press coverage of Apple's statement: MacRumors, 9to5Mac, 2024-03-01).
- Apple's current DMA pages describe alternative browser engines for browser apps and in-app browsing in the EU (iOS 17.4 and later, iPadOS 18). They state no restriction on Home Screen web apps or web push [Apple DMA] [Apple alt engines].
- **Conclusion:** no EU-specific restriction on Home Screen web apps or push is in force as of November 2026. EU Students install and receive push the same way as everyone else.
- One caveat: Apple's support pages do not say whether an EU browser that uses a non-WebKit engine can install web apps with its own engine. That does not affect Cadence, because Safari and WebKit web apps work.

## 5. What is missing from the repo today

| Area | Today | Needed |
| --- | --- | --- |
| `public/` | `favicon.svg` and `images/` only | `sw.js`, `manifest.webmanifest`, PNG icons (192, 512, maskable, 180 apple-touch) |
| `app/root.tsx` | `links` has favicon and fonts. No manifest link, `theme-color` meta, `apple-touch-icon`, or SW registration | Add the manifest and icon links, `<meta name="theme-color">`, and a client-only SW registration effect |
| `vite.config.ts` | `tailwindcss`, `reactRouter`, `tsconfigPaths` | Nothing, if the worker is hand-written in `public/`. Add `vite-plugin-pwa` only for offline support or a bundled TypeScript worker |
| `react-router.config.ts` | `ssr: true` | No change. `react-router-serve` already serves `/sw.js` from the build |
| Server | No `web-push` dependency, no VAPID env vars | `web-push` dependency, env vars, an `/api/push/subscribe` and `unsubscribe` action, a send helper with 404/410 pruning |
| DB | No subscription table | `push_subscriptions` table, migration and seed |
| UI | None | A "Turn on notifications" control in Notification settings, plus the iOS install hint |

A grep for `manifest`, `serviceWorker` and `webmanifest` in `app/` finds nothing. Also, `GLOSSARY.md`, which map #1 refers to, is not committed on `main`.

## 6. Effort estimate (rough)

| Task | Estimate |
| --- | --- |
| Manifest, icons, `root.tsx` links and meta | 0.5 day |
| Service worker (push, click, subscription change) and registration | 0.5–1 day |
| VAPID setup, `push_subscriptions` schema and migration, subscribe/unsubscribe actions, send helper with pruning | 1–1.5 days |
| Permission button, iOS standalone detection and install hint | 0.5–1 day |
| Testing on real devices (iPhone Home Screen app, Android Chrome, desktop Safari/Firefox/Edge), plus an HTTPS tunnel for phones | 0.5–1 day |
| **Total** | **About 3–5 days** |

This excludes Novu integration, scheduling, and the full Notification settings page.

## Not verified in this session

- The exact Safari 26 tap path (More, then Share, then Add to Home Screen). Apple's support guide did not render.
- Whether Novu's push channel can consume the subscriptions stored by Cadence, or wants its own.

## Sources

- [WebKit 13878] Web Push for Web Apps on iOS and iPadOS — https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
- [WebKit 16.4] WebKit Features in Safari 16.4 — https://webkit.org/blog/13966/webkit-features-in-safari-16-4/
- [WebKit 16535] Meet Declarative Web Push — https://webkit.org/blog/16535/meet-declarative-web-push/
- [WebKit WWDC25] News from WWDC25: WebKit in Safari 26 beta — https://webkit.org/blog/16993/news-from-wwdc25-web-technology-coming-this-fall-in-safari-26-beta/
- [WebKit WWDC26] News from WWDC26: WebKit in Safari 27 beta (2026-06-08) — https://webkit.org/?p=17967
- [web.dev subscribing] Subscribing a user — https://web.dev/articles/push-notifications-subscribing-a-user
- [web.dev protocol] The Web Push Protocol — https://web.dev/articles/push-notifications-web-push-protocol
- [MDN requestPermission] https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static
- [MDN installable] Making PWAs installable — https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Making_PWAs_installable
- [MDN display-mode] https://developer.mozilla.org/en-US/docs/Web/CSS/@media/display-mode
- [Vite assets] The public directory — https://vite.dev/guide/assets
- [React Router serve] https://reactrouter.com/api/other-api/serve, plus local source `node_modules/@react-router/serve/dist/cli.js` (v8.3.0)
- [web-push README] https://github.com/web-push-libs/web-push
- [RFC 8030] Generic Event Delivery Using HTTP Push — https://www.rfc-editor.org/rfc/rfc8030
- [Firefox 72 notes] https://www.firefox.com/en-US/firefox/72.0/releasenotes/
- [Apple DMA] https://developer.apple.com/support/dma-and-apps-in-the-eu/
- [Apple alt engines] https://developer.apple.com/support/alternative-browser-engines/
- EU reversal, 2024-03-01 (secondary, quoting Apple): https://www.macrumors.com/2024/03/01/apple-walks-back-decision-to-disable-eu-web-apps/
