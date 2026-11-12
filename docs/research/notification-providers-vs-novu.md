# Knock, Courier, SuprSend, MagicBell and Engagespot compared with Novu

Research for issue #2 (child of map #1, *Notifications for Students and Instructors*). Sources were read on 2026-11-12. Vendor pages change often, so check them again before deciding.

## Question

How do Knock, Courier and SuprSend (plus MagicBell and Engagespot, if time allows) compare with Novu for Cadence? The criteria, in priority order:

1. Open source and self-hostable.
2. EU data residency, plus GDPR for student data (DPA, SCCs, sub-processors).
3. Easy Node and React integration.

Cadence also needs these channels: browser push, an in-app inbox and email. Each person must be able to opt out per Notification type and per channel. The cost has to suit a small course platform.

## Short answer

None of them beats Novu on Cadence's priorities.

- **Open source.** None of the five has an open-source or self-hostable core on a self-serve plan. Their public GitHub repos are SDKs, a CLI and UI kits. Novu's core is MIT.
- **EU data residency.**
  - Knock has none: every listed sub-processor is in the USA, AWS included.
  - Courier has an EU region (Ireland), but only on paid plans.
  - SuprSend's DPA offers an EU workspace region. Its pricing page lists custom regions, self-hosting and BYOC only under Enterprise, so it is unclear whether the free plan gets EU.
  - MagicBell and Engagespot publish nothing verifiable about an EU region.
  - Novu offers an EU (Frankfurt) region on every tier.
- **Free tier.** Knock, Courier and SuprSend are all about 10k messages a month free, which is similar to Novu. MagicBell's free tier is 1k.
- **Browser push.** The best are SuprSend and MagicBell, which both use native VAPID. Knock and Courier reach the browser only through FCM, OneSignal or similar.

The runner-up is **SuprSend**, *if* its EU region turns out to be available below Enterprise. Novu remains the recommendation.

## Comparison

| | **Novu** (baseline, from earlier research) | **Knock** | **Courier** | **SuprSend** | **MagicBell** | **Engagespot** |
|---|---|---|---|---|---|---|
| Open source / self-host | MIT core, self-hostable | No. Only SDKs are public | No | No. Self-hosting and BYOC (beta) are Enterprise only [S3] | No evidence | No evidence |
| EU data residency | EU (Frankfurt) on all tiers | **No.** All 19 sub-processors are in the USA, including AWS hosting [K3] | Yes, Ireland (eu-west-1), on **paid plans only** (Business/Enterprise) [C2] | DPA: "the United States or the European Union" workspace region [S4]. Pricing lists "Custom Data Center Region" under Enterprise [S1]. Plan gating is unclear | Not stated | A "Multi-Region" docs link exists, but no detail could be verified |
| DPA / SCCs | DPA with SCCs | DPA with SCCs Module 2 and UK Addendum [K2] | DPA dated 2026-09-12, SCCs Module 2. Default infrastructure is in the US [C3] | DPA with SCCs Module 2/3, used as a fallback to the EU–US DPF. Support staff may access data from India [S4] | Not found | Not found |
| Free tier | 10k workflow runs/mo | 10k messages/mo. Starter is $250/mo [K1] | 10k sends/mo. Business is $0.005/send [C1] | 10k notifications/mo. Essentials is $100/mo; Business is $250/mo [S1] | 1k deliveries/mo. Startup is $249/mo [M1] | Free plan exists; limits not verified |
| Browser push | Yes | Through FCM, OneSignal, SNS and others only. No native VAPID [K5] | Through FCM, OneSignal and others only. No native VAPID provider is listed [C4] | **Native**: VAPID key plus a service worker, from the React SDK [S2] | **Native** VAPID, with a `WebPushButton` [M2] | Push listed; web push not confirmed |
| In-app inbox for React | Yes (`@novu/react`) | `@knocklabs/react` (feed, popover, bell) [K4] | `@trycourier/courier-react` (`CourierInbox`, popup, toasts) [C5] | React SDK with `SuprSendProvider` [S2] | React inbox components [M2] | UI kit for React |
| Per-type × per-channel preferences | Yes | Yes. `PreferenceSet` covers workflows, categories and channel types; unset means on [K6] | Yes. `CourierPreferences` component and API [C5] | Yes, but preferences are a **Business-plan** feature ($250/mo) [S1] | Yes, as categories × channels with `enabled` [M3] | Yes, "notification preference management" [E1] |

## Findings per vendor

### Knock

- **Data location.** The sub-processor list names AWS as "Cloud hosting and infrastructure — USA". All 19 sub-processors are in the USA [K3]. The DPA authorises international transfers and uses SCCs Module 2 under Irish law [K2]. No Knock page mentions an EU region.
- **Pricing.** The free plan covers 10,000 messages a month, with no hard cap. Starter is $250/month for 50k messages [K1].
- **Integration.** The React SDK is `@knocklabs/react` [K4]. The preference model fits Cadence's opt-out design closely: unset preferences default to `true`, and a Notification sends only if every applicable preference is true [K6].
- **Browser push.** The push providers are APNs, FCM, Expo, Amazon SNS and OneSignal. Web push is not listed as a native channel [K5].
- **Verdict.** Ruled out by the EU data residency priority.

### Courier

- **EU region.** Courier has an EU datacenter in Ireland, "available on every paid plan: Business and Enterprise". It is a separate workspace at `app.eu.courier.com` / `api.eu.courier.com`. "Data you've already sent through the US stays in the US." [C2]
- **DPA.** Last updated 2026-09-12. It says the data center infrastructure is "located in the United States" and uses SCCs Module 2 [C3]. The sub-processor list itself, at courier.com/subprocessors, could not be read.
- **Pricing.** The Developer plan is free for 10k sends a month. Business is $0.005 per send, pay as you go, which also unlocks EU [C1]. At Cadence's scale this is probably a few dollars a month.
- **Integration.** `@trycourier/courier-react` provides an inbox, toasts and a `CourierPreferences` component, authenticated with a JWT from the backend [C5].
- **Browser push.** Only through FCM, OneSignal and other providers. There is no first-party VAPID channel [C4].
- **Verdict.** This is the strongest *hosted, closed-source* alternative with a verified EU region, but it is not free once EU is required. It has no open-source option.

### SuprSend

- **EU region.** The DPA says: "Customer Personal Data is stored in the hosting region of your workspace, the United States or the European Union". Production runs on AWS, GCP and Azure in the US and EU. Staff may access data from India [S4]. The pricing page lists "Multi-region data center", "Custom Data Center Region", "Self-hosted deployment" and BYOC (beta) under **Enterprise** [S1]. The sources do not settle whether a free or Essentials workspace can be created in the EU.
- **Pricing.**
  - Free: 10k notifications a month.
  - Essentials: $100/month.
  - Business: $250/month. This tier adds **user preferences**, batching/digest and i18n [S1].
- **Integration.** Web push is native. The React `SuprSendProvider` takes a `vapidKey`, a service worker at `/serviceworker.js` loads `@suprsend/web-sdk`, and `registerPush()` is called on a user gesture [S2]. The public GitHub org has SDKs, a CLI and web components under MIT. No core engine repo was found [S5].
- **Verdict.** The closest feature match after Novu: native VAPID push, an inbox and preferences. Two things count against it: preferences cost $250/month, and EU on the lower tiers is unconfirmed. It is not open source.

### MagicBell (quick pass)

- **Pricing.** The free Builder plan has 1,000 deliveries a month. Startup is $249/month [M1].
- **Features.** Native web push with VAPID, React inbox components and a `WebPushButton` [M2]. Preferences are stored as categories × channels [M3].
- **Not found.** No EU region, DPA or sub-processor list. Its privacy policy gives a San Francisco address only.
- **Verdict.** Its free tier is too small, and its EU position is unknown.

### Engagespot (quick pass)

- **Not verified.** Its pricing page (engagespot.co/pricing, /pricing.php) returned 404. The docs list a preference manager and an in-app inbox, and a "Multi-Region" docs link exists, but no EU or pricing detail could be verified from primary sources [E1].
- **Verdict.** Not enough verifiable information to recommend it.

## Open questions

1. **SuprSend:** can a free or Essentials workspace be created in the EU region? The DPA and the pricing page disagree in tone. Ask SuprSend support, or try signing up.
2. **Courier:** the current sub-processor list (courier.com/subprocessors) could not be read. Is the EU region truly isolated from the US sub-processors used for analytics and logging?
3. **Engagespot:** what are its pricing and EU region details? Its site returned 404 for the pricing pages.
4. **Novu:** none of these vendors changes the baseline. The decision still rests on Novu's EU Cloud tier, or on self-hosting Novu.

## Sources (read 2026-11-12)

- [K1] Knock pricing — https://knock.app/pricing
- [K2] Knock DPA — https://knock.app/legal/data-processing-addendum
- [K3] Knock sub-processors — https://knock.app/legal/subprocessors
- [K4] Knock React in-app UI — https://docs.knock.app/in-app-ui/react/overview
- [K5] Knock push integrations — https://docs.knock.app/integrations/push/overview
- [K6] Knock preferences — https://docs.knock.app/preferences/overview
- [C1] Courier pricing — https://www.courier.com/pricing
- [C2] Courier EU datacenter — https://www.courier.com/docs/platform/workspaces/eu-datacenter
- [C3] Courier DPA — https://www.courier.com/data-processing-addendum
- [C4] Courier push providers — https://www.courier.com/docs/external-integrations/push/intro-to-push
- [C5] Courier React SDK — https://www.courier.com/docs/platform/inbox/web/react-sdk/
- [S1] SuprSend pricing — https://www.suprsend.com/pricing
- [S2] SuprSend React web push — https://docs.suprsend.com/docs/react-webpush
- [S3] SuprSend security — https://docs.suprsend.com/docs/security
- [S4] SuprSend DPA — https://www.suprsend.com/data-processing-addendum
- [S5] SuprSend GitHub org — https://github.com/suprsend
- [M1] MagicBell pricing — https://www.magicbell.com/pricing
- [M2] MagicBell React components and web push channel — https://www.magicbell.com/docs/libraries/magicbell-react, https://www.magicbell.com/docs/channels/web_push
- [M3] MagicBell user preferences — https://www.magicbell.com/docs/api/reference
- [E1] Engagespot docs — https://docs.engagespot.co/docs/introduction/getting-started/
