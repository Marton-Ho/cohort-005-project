# Novu integration check: React Inbox, Node SDK, web push, preferences, EU region

Research for #3 (map #1, "Notifications for Students and Instructors"). Checked on 2026-11-12 against npm registry metadata, docs.novu.co and the `novuhq/novu` and `novuhq/novu-ts` GitHub repos.

Cadence stack, from `package.json`: React 19.2, React Router 8.3 (SSR, Vite 7), Node, zod 3.25.

## TL;DR

| Question | Answer |
| --- | --- |
| `@novu/react` on React 19 + RR v8 SSR | **Yes.** Peer deps allow React 19. The package ships a separate server build (a stub), so SSR does not crash. Mount `<Inbox>` client-only to avoid a hydration mismatch. |
| Node SDK | **`@novu/api`** (Speakeasy-generated, fully typed, zod-validated, retries, about monthly releases). `@novu/node` is deprecated. `@novu/framework` is optional (code-first workflows). |
| Direct VAPID web push | **No native VAPID provider.** The options are FCM, OneSignal, Pusher Beams, Pushpad, APNs, Expo, AppIO, or **Push Webhook**. For standard VAPID (including iOS 16.4+ Home Screen web apps), Cadence uses Push Webhook and sends with `web-push` itself. |
| Preferences | Per-workflow × per-channel booleans, plus global. One workflow per Notification type gives the per-type, per-channel opt-out directly. Cadence's own settings UI can drive it through `PATCH /v2/subscribers/{id}/preferences`. |
| EU region | **Yes.** `serverURL: "https://eu.api.novu.co"` (SDK), and `backendUrl="https://eu.api.novu.co"` plus `socketUrl="wss://eu.socket.novu.co"` (Inbox). |

## 1. `@novu/react` (Inbox, Bell) with React 19 and React Router v8 SSR

- Latest is `@novu/react@3.19.2`. Its peer deps are `react: "^18.0.0 || ^19.0.0 || ^19.0.0-0"` and the same for `react-dom` (`npm view @novu/react peerDependencies`). The docs say it "requires React 18 or later" ([get-started](https://docs.novu.co/platform/inbox/react/get-started)).
- Its only runtime dependency is `@novu/js@3.19.2` (`npm view @novu/react dependencies`).
- **SSR behaviour.** The `exports` map sends the plain `import`/`require` conditions (Node, and Vite's SSR build) to `dist/esm/server/index.js`. Only the `browser` condition gets the real build (`npm view @novu/react exports`). In the server build, `Inbox` renders only a `ShadowRootDetector`, and `Bell`, `InboxContent`, `Notifications` and `Preferences` are empty functions ([unpkg: server/index.js](https://unpkg.com/@novu/react@3.19.2/dist/esm/server/index.js)). Importing `@novu/react` in a route module therefore will not crash React Router's server render, but server and client markup differ.
- **What Cadence should do.** Render `<Inbox>` only after mount (a small `ClientOnly`/`useEffect` gate, or `React.lazy` plus `Suspense` with a bell-shaped fallback). This avoids hydration warnings. Novu's own Next.js template marks the component `'use client'`, which confirms it is a client-only widget ([get-started](https://docs.novu.co/platform/inbox/react/get-started)).
- **Routing.** Pass `routerPush={(path) => navigate(path)}` with React Router's `useNavigate` so that clicking a notification does client-side navigation ([get-started](https://docs.novu.co/platform/inbox/react/get-started)).
- **Auth.** Turn on HMAC in the In-App provider settings and pass `subscriberHash = HMAC-SHA256(NOVU_SECRET_KEY, subscriberId)` as lowercase hex, computed in a loader (server-only). "If HMAC encryption is active … and `subscriberHash` is not provided, the Inbox will not load" ([prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production)). This fits Cadence's loader model: compute the hash in the root or layout loader and pass it down.

## 2. Node SDK: `@novu/api` (and `@novu/framework`)

- **`@novu/node` is deprecated.** "March 20, 2025 marks the end of support for @novu/node. Users can migrate to the new @novu/api package" (`npm view @novu/node deprecated`).
- **`@novu/api`** (latest 3.19.1, 2026-09-02; repo `novuhq/novu-ts`):
  - It is "SDK Created by Speakeasy", generated from the OpenAPI spec, so its types follow the API ([README](https://github.com/novuhq/novu-ts/blob/main/README.md)). It ships `index.d.ts`. Its only dependency is `zod: "^3.25.0 || ^4.0.0"`, which is compatible with Cadence's zod 3.25 (`npm view @novu/api dependencies`).
  - It has configurable retries (`retryConfig`, `strategy: "backoff"`), a `NovuError` base class for HTTP errors, a `ResponseValidationError` class, and standalone tree-shakeable functions ([README](https://github.com/novuhq/novu-ts/blob/main/README.md)).
  - It runs on Node, Bun, Deno and Cloudflare Workers ([TS SDK docs](https://docs.novu.co/platform/sdks/server/typescript)).
  - **Release cadence:** 3.16.0 (May 13), 3.17.0 (Jun 1), 3.17.1 (Jul 7), 3.18.0 to 3.18.2 (Jul 8 to 19), 3.19.0 (Aug 7), 3.19.1 (Sep 2), all in 2026 (`npm view @novu/api time`). That is roughly one release a month. Nothing has shipped since early September, which is a slight slowdown but not a concern. The main `novuhq/novu` repo was last pushed 2026-11-11 and has about 40k stars (`gh api repos/novuhq/novu`).
  - Usage is `new Novu({ secretKey, serverURL: "https://eu.api.novu.co" })`, then `novu.trigger({ workflowId, to: { subscriberId, email }, payload })` ([TS SDK docs](https://docs.novu.co/platform/sdks/server/typescript)).
- **`@novu/framework`** (2.14.0) is optional. It defines workflows in code and serves them from a "Bridge Endpoint" (`/api/novu`) in your app, which Novu Cloud calls ([framework overview](https://docs.novu.co/framework/overview)). Adapters exist for Express, Remix, H3, Hono, Next.js, NestJS and others. Its peer deps are all optional framework adapters (`npm view @novu/framework peerDependencies`). There is no React Router v8 adapter by name. The Remix, Express or H3 adapter would need checking. Cadence can skip it and define workflows in the dashboard, triggering them with `@novu/api` only.

## 3. Web push: direct VAPID or through a push provider?

- **Novu has no native Web Push (VAPID) provider.** The push providers in source are `apns`, `appio`, `expo`, `fcm`, `one-signal`, `push-webhook`, `pusher-beams` and `pushpad` ([`packages/providers/src/lib/push`](https://github.com/novuhq/novu/tree/next/packages/providers/src/lib/push)). The docs list the same set ([push integrations](https://docs.novu.co/platform/integrations/push)). The feature request "Add Web Push-API as a Push Provider for those that only build web apps" is still **open** ([novuhq/novu#3459](https://github.com/novuhq/novu/issues/3459)).
- **Options for browser push:**
  1. **FCM webpush.** Novu supports FCM `webpush` overrides and `fcmOptions.link` ([FCM](https://docs.novu.co/platform/integrations/push/fcm)). This adds Firebase (a Google dependency, outside the EU-only story) and the Firebase JS SDK in the service worker.
  2. **Pushpad, OneSignal or Pusher Beams.** These are third-party browser push services. The subscriber's provider `uid` is registered as Novu `deviceTokens` ([Pushpad](https://docs.novu.co/platform/integrations/push/pushpad)). Each adds another vendor and data processor.
  3. **Push Webhook (recommended for Cadence).** "Novu sends push notifications directly to a webhook URL that you control." The POST body has `target` (the subscriber's device tokens), `title`, `content`, `messageId`, `overrides` and `payload`. Requests are signed with `x-novu-signature`, which is HMAC-SHA256 of the body using your secret. You can choose the device token string yourself ([Push Webhook](https://docs.novu.co/platform/integrations/push/push-webhook)). Retry behaviour is **not documented**.
- **iOS Home Screen web apps** get standard W3C Web Push from iOS/iPadOS 16.4: "the same W3C standards-based Web Push that was added in Safari 16.1". No Apple Developer Program membership is needed. Permission must be requested "in response to direct user interaction", and only from an app added to the Home Screen ([WebKit blog](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)). A plain VAPID sender (the `web-push` npm package) works for iOS, so FCM is not needed.

**What Cadence would need for VAPID through Push Webhook:**

1. Generate a VAPID key pair and keep the private key in server env.
2. Add a PWA manifest (`display: standalone`) and a service worker with `push` and `notificationclick` handlers.
3. Add a "Turn on browser notifications" button that calls `PushManager.subscribe({ applicationServerKey })` and POSTs the `PushSubscription` to Cadence.
4. Add a `pushSubscriptions` table (user, endpoint, p256dh, auth, created, last success) in SQLite/Drizzle.
5. Register one opaque device token per subscription (for example the row id) on the Novu subscriber's Push Webhook credentials (`deviceTokens`).
6. Add a webhook route. It verifies `x-novu-signature`, looks up each `target` token, sends with `web-push`, deletes the subscription on a 404/410 response, and optionally reports `delivered`/`failed` back to Novu.

Cadence owns the subscription storage, the VAPID sending and the expiry cleanup either way. Novu adds workflow, preference and digest handling on top.

## 4. Subscriber preferences and per-type, per-channel opt-outs

- **Model** ([preferences concept](https://docs.novu.co/platform/concepts/preferences)), from least to most specific:
  1. Workflow channel defaults (in code or the dashboard; all channels on by default).
  2. Subscriber global preferences.
  3. Subscriber per-workflow channel preferences.

  "The most specific layer that defines a channel wins." **Critical** workflows ignore subscriber preferences and are hidden from the preferences UI.
- **API.** `PATCH /v2/subscribers/{subscriberId}/preferences` with a body of `{ workflowId?, channels: { email, sms, in_app, push, chat, tool }, schedule? }`. Leaving out `workflowId` updates global preferences. SDK call: `novu.subscribers.preferences.update({ workflowId, channels }, subscriberId)` ([API reference](https://docs.novu.co/api-reference/subscribers/update-subscriber-preferences)). A matching "retrieve subscriber preferences" endpoint exists.
- **Mapping to Cadence.** Use one Novu workflow per Notification type (`streak`, `comeback`, `comment`). Each type × channel switch in Cadence's settings then maps to exactly one boolean in `channels` for that `workflowId`. Opt-out by default matches Novu's "all enabled" default. The optional `schedule` (weekly hours) could cover quiet hours later.
- **Can Cadence's own settings UI drive it?** Yes. A settings route action calls `subscribers.preferences.update` from the server. Alternatively, the client can use the `usePreferences` hook or the standalone `<Preferences>` component. The Inbox's built-in preferences view can be filtered (`preferencesFilter`, `preferenceGroups`) or hidden per workflow through `appearance`. Global preferences "are always shown … and cannot be removed" except by hiding them with `appearance` ([Inbox preferences](https://docs.novu.co/platform/inbox/configuration/preferences)). If Cadence keeps the source of truth in its own DB (an open question in map #1), it must write through to Novu on every change. It must also hide the Inbox's own preferences view, or users could change settings in two places.
- **Caveats.**
  - Novu preferences are per channel, not per provider. The Push Webhook is one `push` channel, so "browser push off" is a single switch.
  - The "at most one Notification a day across all courses" rule is not a preference. Cadence or a Novu digest/throttle step would have to enforce it.

## 5. EU region

- REST/SDK: `serverURL: "https://eu.api.novu.co"` ([TS SDK docs](https://docs.novu.co/platform/sdks/server/typescript)).
- Inbox: `backendUrl="https://eu.api.novu.co"` and `socketUrl="wss://eu.socket.novu.co"` ([get-started](https://docs.novu.co/platform/inbox/react/get-started)).
- Not verified from a primary source: whether the EU region is a separate account or dashboard (keys and workflows not shared with US). Assume it is when setting up.

## Open questions

- Push Webhook retry and timeout behaviour is undocumented. Test it, or ask Novu.
- Whether `@novu/framework`'s Remix, Express or H3 adapter mounts cleanly as a React Router v8 resource route (only relevant if Cadence picks code-first workflows).
- Whether the Inbox's `ShadowRootDetector` SSR output avoids hydration warnings without a client-only gate. Assume it does not, and gate it.
- Free-tier limits and pricing for EU Cloud were not checked here.
