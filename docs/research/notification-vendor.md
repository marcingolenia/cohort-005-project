# Which third-party service delivers Notifications?

Research for issue #3, a child of map #1 (Wayfinder: Notification system spec).
All sources were accessed on **2026-11-19**. Vendor pages change often, so re-check prices before signing up.

## Context and assumptions

- Cadence is a React Router SSR app (`react-router` 8.3.0 and React 19.2 in `package.json`). It stores data in one SQLite file through Drizzle (ADR 0006).
- Today `app/lib/mailer.server.ts` sends plain-text email through the Postmark `/email` API, using the `outbound` stream. In dev and tests it writes to `.outbox/` instead. `sendEmail` has two callers: `admin.refunds.tsx` and `courses.$slug.refund.ts`. A welcome email is stubbed in `enrollUser`.
- Only email and in-app channels are in scope. Muting and preferences are out of scope, so they carry no weight here.
- Scale assumption: a few thousand users and about **20k Notification events a month**. Each event goes to email, in-app, or both, so assume up to 40k channel messages.

## Summary table

| | Knock | Novu (Cloud) | Courier | Postmark + home-built inbox |
|---|---|---|---|---|
| In-app React component | `@knocklabs/react` feed, popover and bell, with WebSocket real-time [K2] | `@novu/react` `<Inbox>`, real-time, React 18/19 [N3] | `@trycourier/courier-react`, renders client-side only [C3] | Your own component, rendered in the loader on the server |
| SSR fit | Client-side provider. The docs say nothing about SSR [K2] | Examples mark it `'use client'`. Its example uses React Router `useNavigate` [N4] | Explicitly client-side only [C3] | Native: server-rendered and revalidated by loaders and actions |
| Keeps Postmark for email | Yes, Postmark channel [K1] | Yes, Postmark integration [N5] | Yes, Postmark integration with overrides [C4] | Yes, it is already in place |
| Digests and batching | Strong batch function: fixed, sliding or dynamic windows, batch keys. Templates see 10 activities, 100 on Enterprise only [K3] | Digest step with regular or scheduled windows and a key. Max window is 1 day on Free and 7 days on Pro [N6] | Batch and digest nodes in Journeys. Digests need two automations [C5] | Build it yourself: a table plus a scheduled job |
| Billing unit | Per message per channel (email + in-app = 2) [K4] | Per workflow run, whatever the channel count [N7] | Per send attempt, counted per recipient. Includes failures and tests [C6] | Postmark emails only |
| Est. cost at about 20k events/month | ~40k messages is above the 10k free tier, so Starter at **$250/mo** [K5] | 20k runs fits **Pro, from $30/mo** (30k runs). Free covers 10k [N1] | Business at $0.005/send: **~$100/mo**, or up to ~$200 if Journeys bill per Send node [C1][C6] | Postmark alone: **~$15–40/mo** for 10–25k emails [P1] |
| EU data residency | None found. All subprocessors are in the USA [K6] | EU region (`eu.api.novu.co`), listed on Free [N1][N4] | EU datacenter in Ireland, Business and Enterprise plans [C2] | SQLite stays wherever Cadence is hosted. Postmark is US-only [P2] |
| Self-hosting | No | Community edition: Inbox and digest included, 1 team member, needs MongoDB and 2 Redis [N8][N9] | No | Already self-hosted |
| Effort to migrate the existing emails | Move 2–3 emails into Knock templates and swap `sendEmail` for a workflow trigger | Same, plus the subscriber/HMAC setup | Same, plus JWT minting | Close to none: keep `sendEmail` behind a new `notify()` |

## Findings per option

### Knock

- **Email:** Postmark is a first-class channel. You supply the server token and From address, and you can set payload overrides such as `MessageStream`. Delivery webhooks report `delivered` and `bounced` [K1].
- **In-app:** `@knocklabs/react` provides `KnockProvider`, `KnockFeedProvider`, `NotificationFeedPopover` and `NotificationIconButton`, plus headless hooks. Real-time runs over a socket [K2]. Production use needs enhanced security mode, where the server signs an RS256 user token [K7]. The docs do not cover SSR, so the feed renders after hydration.
- **Batching:** This is the most capable batching of the four. It offers fixed, dynamic, relative and sliding windows, batch keys and a max-activity close. Templates see `total_activities` and up to 10 `activities`; seeing up to 100 is Enterprise only [K3].
- **Pricing:** Free covers 10k messages a month. Starter costs $250/mo with 50k messages, then $0.005 per message [K5]. Knock bills per user per channel, so email plus in-app counts as two [K4]. At Cadence's scale this lands on Starter almost immediately.
- **EU:** The subprocessors list gives USA as the location for every subprocessor, including AWS hosting. I found no EU region [K6].

### Novu

- **Email:** Postmark is a supported email integration [N5].
- **In-app:** The `@novu/react` `<Inbox>` supports React 18 and 19, with a real-time unread count [N3]. The setup docs mark the Next.js example `'use client'`. The React example passes React Router's `useNavigate` into `routerPush`. For the EU region you set `backendUrl="https://eu.api.novu.co"` and `socketUrl="wss://eu.socket.novu.co"` [N4]. Updating subscriber profiles from the client requires an HMAC `subscriberHash` [N3].
- **Digest:** The digest step groups by subscriber plus an optional key. It has regular windows, which can start immediately or when events repeat, and scheduled windows. The max window is 1 day on Free, 7 on Pro, 30 on Team [N6].
- **Pricing:** Free gives 10k workflow runs a month and 5 workflows. Pro starts at $30/mo with 30k runs and 20 workflows, then $1.20 per extra 1k runs [N1]. One run per subscriber covers every step and channel in the workflow, so email plus in-app counts once [N7]. Of the three vendors this is the cheapest at Cadence's scale.
- **EU and self-hosting:** The pricing page lists EU data residency on Free [N1]. The self-hosted Community edition includes digest and the Inbox components. It is limited to 1 team member and lacks snooze, branding removal and email activity tracking [N9]. It needs MongoDB, two Redis instances (one with AOF) and object storage, so it is far heavier than Cadence's single SQLite file [N8].

### Courier

- **Email:** Postmark is supported. You can override `MessageStream`, use Postmark templates and fail over to a second provider [C4].
- **In-app:** `@trycourier/courier-react` is for React 18+. The docs say the Inbox and Toast "support Next.js but render client-side only" [C3]. The server mints a JWT with inbox scopes, and the client calls `signIn` and `listenForUpdates()` [C3].
- **Batching:** Journeys provide batch and digest nodes. Batch releases on inactivity, max count or max wait, with the max wait capped at 60 days. A digest needs one automation to collect events and a second to release them [C5].
- **Pricing:** The Developer plan is free up to 10k sends. Business costs $0.005 per send, starting from the first send, and Enterprise is custom [C1]. Courier bills per attempt, so failed, test and preference-blocked messages all count [C6].
- **EU:** A separate datacenter in Ireland is available on Business and Enterprise. It has its own API (`api.eu.courier.com`) and Inbox hosts (`inbox.eu.courier.io`, `realtime.eu.courier.io`) [C2].

### Postmark + a home-built in-app inbox

- **Email:** Keep what exists. Postmark costs $15/mo on Basic or $16.50 on Pro for 10k emails, plus $1.20–1.80 per extra 1k [P1].
- **In-app:** Add a `notifications` table in SQLite: recipient, type, payload as JSON, `readAt`, `createdAt`. A root or layout loader returns the unread count and recent items, and an action marks items read. This is the only option that renders on the server with no client SDK, socket or token minting. React Router revalidation refreshes it after any action, and polling with `useRevalidator` or `useFetcher` can stand in for real-time.
- **Batching:** You build it: a "pending digest" row per recipient and key, flushed by a scheduled job. This is the main build cost. Map #1 only says digests "depend on the catalogue", so there may be little to build.
- **EU:** In-app data stays in Cadence's own database. Postmark itself stores data in the US (Chicago-area datacenter plus AWS) and has no EU region [P2]. **This applies to every option**, because all three vendors would still hand email to Postmark.
- **Reliability:** You also own retries and moving sends out of the request, an open question on map #1. The vendors give you this for free.

## Recommendation

**Use Postmark + a home-built in-app inbox.** Keep Novu Cloud (EU region, Pro) as the fallback if digests become complex.

Reasons:

1. **SSR fit:** All three vendor components render on the client only, behind a socket and a signed token. A loader-rendered inbox matches how the rest of Cadence works and needs no extra secrets.
2. **Scale and cost:** At about 20k events a month the vendors cost $30–250/mo for features Cadence will barely use. Preferences, the vendors' other big selling point, are out of scope.
3. **Migration:** Close to zero. Wrap the existing `sendEmail` and the `.outbox/` transport (which tests already use) in a single `notify()` that writes the in-app row and sends the email.
4. **Data residency:** In-app data stays in Cadence's SQLite. The vendors would add a second processor that also sees the data. Knock has no EU region at all.

Trade-offs to accept:

- **Digests and batching** must be built and tested in-house. This is the strongest argument for a vendor, and Knock's batching is the most capable.
- **No real-time push** unless SSE is added later. Polling and revalidation should be enough for a course platform.
- **Delivery reliability** is yours to build: a background queue, retries, and Postmark bounce webhooks.
- **Editing templates without developers** isn't possible, because templates stay in code.

**Switch to Novu if** the Notification catalogue (map #1) ends up needing several kinds of digest or real-time feeds. It bills one run per event across channels, has an EU region from Free, keeps Postmark, and can be self-hosted as an exit route. Knock fits batching best but costs the most and lacks EU residency. Courier sits in between and bills failed sends.

## Sources (accessed 2026-11-19)

- [K1] Knock: Postmark integration — https://docs.knock.app/integrations/email/postmark
- [K2] Knock: React in-app UI overview / feed — https://docs.knock.app/in-app-ui/react/overview, https://docs.knock.app/in-app-ui/react/feed
- [K3] Knock: Batch function — https://docs.knock.app/designing-workflows/batch-function
- [K4] Knock: Plans and usage (what counts as a message) — https://docs.knock.app/manage-your-account/knock-plans
- [K5] Knock: Pricing — https://knock.app/pricing
- [K6] Knock: Subprocessors — https://knock.app/legal/subprocessors
- [K7] Knock: In-app security and authentication — https://docs.knock.app/in-app-ui/security-and-authentication
- [N1] Novu: Pricing — https://novu.co/pricing
- [N3] Novu: Inbox overview — https://docs.novu.co/platform/inbox/overview
- [N4] Novu: Set up the Inbox (EU URLs, React Router example) — https://docs.novu.co/platform/inbox/setup-inbox
- [N5] Novu: Postmark integration — https://docs.novu.co/platform/integrations/email/postmark
- [N6] Novu: Digest step — https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest
- [N7] Novu: Billing (workflow run definition) — https://docs.novu.co/platform/account/billing
- [N8] Novu: Self-hosting overview — https://docs.novu.co/community/self-hosting-novu/overview
- [N9] Novu: Self-hosted vs Novu Cloud — https://docs.novu.co/community/self-hosted-and-novu-cloud
- [C1] Courier: Pricing — https://www.courier.com/pricing
- [C2] Courier: EU datacenter — https://www.courier.com/docs/platform/workspaces/eu-datacenter
- [C3] Courier: React SDK — https://www.courier.com/docs/platform/inbox/web/react-sdk
- [C4] Courier: Postmark integration — https://www.courier.com/docs/external-integrations/email/postmark
- [C5] Courier: Batching / Digests — https://www.courier.com/docs/platform/automations/batching, https://www.courier.com/docs/platform/automations/digest
- [C6] Courier: Workspaces and billing notes — https://www.courier.com/docs/workspaces/overview
- [P1] Postmark: Pricing — https://postmarkapp.com/pricing
- [P2] Postmark: EU data protection / GDPR FAQ — https://postmarkapp.com/eu-privacy, https://postmarkapp.com/support/article/1218-gdpr-faq

Caveats: Courier's per-recipient billing rule, and whether Journeys bill per Send node, came from search summaries, not a pricing page that states it outright. Confirm with Courier before relying on the ~$100 figure. The Novu socket host has also appeared as `eu.ws.novu.co` in older docs.
