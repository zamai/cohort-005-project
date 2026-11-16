# Web Push feasibility for Cadence

Research for issue #2 (child of the map #1, *Engagement notifications*). This file reports facts only and makes no product decisions. Research date: 2026-11-16.

**Context:** Cadence is a React Router 8.3 SSR app (`ssr: true`) on Node. It runs on a single server with SQLite through Drizzle. It is web-only, so "mobile" here means Web Push to a mobile browser or an installed PWA.

**Legend:** ✅ verified against a primary source (spec text, MDN browser-compat-data JSON, Apple's doc JSON, package source). ⚠️ uncertain, secondary, or inferred.

---

## 1. Browser and OS support today

The support table comes from MDN `@mdn/browser-compat-data` **8.1.9** (built 2026-11-12), queried directly. ✅

| Feature | Chrome (desktop) | Chrome Android | Firefox | Safari macOS | Safari iOS/iPadOS | Samsung Internet |
|---|---|---|---|---|---|---|
| `PushManager` / `subscribe()` | 42 | 42 | 44 (Android 48) | 16 (macOS 13 Ventura+) | **16.4, Home Screen web apps only** | 4.0 |
| `ServiceWorkerRegistration.showNotification` | 42 | 42 | 44 | 16 | 16.4 | 4.0 |
| `Notification.requestPermission()` | 20 | 42 | 22 | 15 | 16.4 (partial, see below) | 4.0 |
| `PushSubscription.expirationTime` | 60 | 60 | 96 | 16 | 16.4 | 8.0 |
| `pushsubscriptionchange` event | **138** | **138** | 44 (partial) | 16 | **not supported** | 30.0 |
| `window.pushManager` (Declarative Web Push) | no | no | no | 18.4 | 18.4 | no |
| Badging `navigator.setAppBadge` | 81 | no | no | 17 | 16.4 | no |

The BCD notes say the following, quoted verbatim:
- Safari iOS, `PushManager.subscribe`: "Notifications are supported in web apps saved to the home screen."
- Safari iOS, `Notification.requestPermission`: "The parent `Notification` interface is undefined unless the page is a web app saved to the home screen. The app's manifest must have a non-default `display` value."
- Firefox `subscribe` and `requestPermission`: "From Firefox 72 onwards, can only be called in response to a user gesture such as a `click` event."
- Chrome `subscribe`: "The `options` parameter with a `applicationServerKey` value is required."
- Firefox `pushsubscriptionchange`: "The event does not have the `oldSubscription` and `newSubscription` properties" (bug 1497429).

**Apple:** Web push works in "Home Screen web apps in iOS 16.4 or later and Webpages in Safari 16 for macOS 13 or later". It does not require Apple Developer Program membership. ✅ [Apple: Sending web push notifications in web apps and browsers](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)

**Third-party iOS browsers:** Starting with iOS 16.4, third-party browsers can offer Add to Home Screen. A web app added that way gets the same push capability. ✅ [WebKit: Web Push for Web Apps on iOS and iPadOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)

## 2. Does iOS require an installed Home Screen web app?

**Yes, on iOS and iPadOS.** In mobile Safari, a plain tab has no `PushManager` and no `Notification`. Push is available only after the user has added the site to the Home Screen and opens it from there. ✅ (BCD notes above, WebKit 16.4 post, Apple doc.)

**Change in iOS/iPadOS 26:** According to WebKit, every site added to the Home Screen now opens as a web app by default, and "Giving users a web app experience simply no longer requires a manifest file". The add sheet has an "Open as Web App" toggle, which is on by default. A manifest is still used if one is present, including its icons. ✅ [WebKit Features in Safari 26.0](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/)

⚠️ **Unresolved conflict:** The current BCD note still says the manifest must have a non-default `display` for `Notification` to exist. I found no primary source that says whether push works on iOS 26 for a site that was added to the Home Screen *without* a manifest. Shipping a manifest with `display: "standalone"` meets both readings.

**Install friction:** Installing is a manual flow (Share → Add to Home Screen). I found no install-prompt API for iOS in the primary sources I checked. ⚠️ (absence of evidence)

**Android and desktop:** Push works in an ordinary browser tab. Installing is not required. ✅ (BCD has no install note for Chrome, Firefox or Samsung.)

⚠️ **Not researched:** Android 13+ also has an OS-level notification permission (`POST_NOTIFICATIONS`). I did not check how Chrome on Android surfaces it.

⚠️ **Not verified:** The status of Home Screen web apps in the EU under the DMA, where Apple announced a removal in iOS 17.4 beta and then reversed it. I did not find a current primary Apple page that confirms it.

## 3. Making Cadence installable / push-capable

**Requirements:**
- **HTTPS:** Push and notifications are "available only in secure contexts (HTTPS)". ✅ [MDN PushManager.subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe). Note that `localhost` counts as a secure context for development. ⚠️ (general web-platform rule; not re-verified for this note)
- **Service worker:** You need one to receive `push` events and call `showNotification`. ✅ (Apple doc, MDN). Safari 18.4+ Declarative Web Push is the exception; see §5.
- **Chrome install criteria** ✅ [web.dev install criteria](https://web.dev/articles/install-criteria):
  - HTTPS, and the app is not already installed.
  - A manifest with `name` or `short_name`, 192px and 512px icons, `start_url`, and `display` set to one of `fullscreen`, `standalone`, `minimal-ui` or `window-controls-overlay`.
  - `prefer_related_applications` absent or false.
  - An engagement heuristic: at least one click or tap, and at least 30 seconds of viewing.
  - The page does not list a service-worker fetch handler as a requirement.
- **iOS:** Has "zero requirements for 'installability'" in Safari 26. On 16.4–18.x, a manifest with `display: standalone|fullscreen` (or the legacy web-app-capable meta tag) is what makes the shortcut a web app rather than a bookmark. ✅ (WebKit 16.4 and 26.0 posts)

**This repo:**
- `public/` holds only `favicon.svg` and `images/`.
- `app/root.tsx` already exports `links` and renders `<Meta/>`/`<Links/>`.
- There is no manifest and no service worker yet. ✅ (repo inspection)

**Serving the files in React Router 8:** There are two options. ⚠️ (inferred, not tested)
1. Put static files in `public/` (for example `public/sw.js` and `public/manifest.webmanifest`). The Vite build serves them at the root path. Add `<link rel="manifest">` through the `links` export.
2. Use a **resource route**, which is a route module that exports a `loader` and no default component and returns a `Response` with custom headers. ✅ [React Router: Resource Routes](https://reactrouter.com/how-to/resource-routes)

A service worker served at `/sw.js` gets root scope by default. Option 2 lets you set `Content-Type` and `Cache-Control` explicitly for `sw.js`. ⚠️ (scope rule not re-verified here)

## 4. VAPID and the Node push library

**VAPID (RFC 8292)** ✅ [RFC 8292](https://www.rfc-editor.org/rfc/rfc8292.html):
- **Token:** An ES256 (ECDSA P-256) signed JWT.
  - `aud` = origin of the push service.
  - `exp` must be no more than 24 h ahead.
  - `sub` is optional and SHOULD be a `mailto:` or `https:` contact.
- **Header:** `Authorization: vapid t=<jwt>, k=<base64url public key>`.
- **Key binding:** A subscription created with `applicationServerKey` is bound to that public key. Every push to it must be signed by the matching private key. **Rotating the VAPID key requires new subscriptions.** The server must remember which key each subscription used (§4.2).
- **Key separation:** The VAPID key must differ from the encryption ECDH key. Push services SHOULD reject a request that uses the same key for both.

**Apple's rules** ✅ (Apple doc):
- **Public key match:** The key in `Authorization` must match the one given to `PushManager.subscribe`. A mismatch returns `VapidPkHashMismatch`.
- **JWT refresh:** "Don't refresh your JWT more frequently than once per hour."
- **Subject:** A JWT whose subject isn't a URL or `mailto:` gets `BadJwtToken`. The web-push README adds that Apple may reject a localhost-based subject with `403 BadJwtToken`.
- **Network:** If an egress allowlist applies, allow `https://*.push.apple.com`. TLS with SNI is required, and both HTTP/1.1 and HTTP/2 are supported.

**`web-push` (npm)** ✅ [web-push-libs/web-push](https://github.com/web-push-libs/web-push), checked with the npm registry and the 4.0.0 tarball source:
- **Releases:**
  - Latest is **4.0.0, published 2026-11-06**. The previous release was 3.6.7 on 2024-01-16, so no release shipped for about 2¾ years.
  - The 4.0.0 notes include: ESM migration, class syntax, Playwright and Firefox CI, dependency refreshes, and a docs warning against localhost VAPID subjects.
- **Activity:** The repo is not archived. The last push was 2026-11-10, there are 44 open issues, and Dependabot PRs are being merged.
- **Package:**
  - `"type": "module"` with `main: src/index.js`, so **4.x is ESM**. Cadence's `package.json` is also `"type": "module"`.
  - `engines.node >= 16`.
  - License is MPL-2.0.
  - Dependencies: `jws`, `asn1.js`, `http_ece` 1.2.1, `https-proxy-agent`.
- **Types:** The 4.0.0 tarball ships **no TypeScript declarations**, and `@types/web-push` is still at **3.6.4**. ⚠️ I did not check whether the 3.x types match the 4.x named-export API.
- **API:**
  - Exports: `setVapidDetails`, `sendNotification`, `generateRequestDetails`, `generateVAPIDKeys`, `getVapidHeaders`, `encrypt`, `WebPushError`.
  - Defaults: `TTL` = 2,419,200 s (4 weeks), `contentEncoding` = `aes128gcm`, `urgency` = `normal`.
  - `topic` is validated as at most 32 URL-safe base64 characters.
  - VAPID JWT `exp` defaults to 12 h, with a maximum of 24 h.
- **Errors:** A non-2xx response rejects with a `WebPushError` carrying `statusCode`, `headers`, `body` and `endpoint`. The library does not delete or retry anything on 404/410 itself; the caller has to handle those.
- ⚠️ **JWT signing:** `generateRequestDetails` calls `getVapidHeaders` on every send, which signs a fresh JWT per request. Apple asks for at most one refresh per hour. I found no source saying whether Apple enforces this or what happens if a sender exceeds it.

## 5. Payload size, encryption, and delivery headers

**Payload encryption** ✅ [RFC 8291](https://www.rfc-editor.org/rfc/rfc8291.html):
- **Scheme:** Encryption is mandatory, using `aes128gcm` (RFC 8188) with ECDH P-256 and a 16-octet `auth` secret per subscription.
- **Single record:** The message is sent as one record. Compressing content encodings are forbidden.
- **Size limit:** Verbatim from §4: "A push service is not required to support more than 4096 octets of payload body … this equates to, at most, 3993 octets of plaintext."
- **Rejection:** Per RFC 8030 §7.2, a push service must not return 413 for 4096 bytes or less.
- **Apple's limit:** Apple returns `413 PayloadTooLarge` for payloads "over the limit of 4 KB". ✅

**Subscription object:** A `PushSubscription` contains:
- `endpoint`: the push service URL, which is a capability URL.
- `expirationTime`: may be `null`.
- `options`.
- keys `p256dh` and `auth`, obtained with `getKey()` or `toJSON()`.

The server must store at least the endpoint and both keys for each subscription. ✅ [MDN PushSubscription](https://developer.mozilla.org/en-US/docs/Web/API/PushSubscription)

**Delivery headers (RFC 8030)** ✅ [RFC 8030](https://www.rfc-editor.org/rfc/rfc8030.html):
- **`TTL`** (§5.2):
  - Required; a request without it gets 400.
  - The push service may shorten the TTL and reports the value it applied.
  - With `TTL: 0`, the message is delivered only if the device is reachable right then; otherwise it is dropped.
  - Apple stores messages "for 30 days or fewer" and limits how many it queues for an offline device.
- **`Urgency`** (§5.3): `very-low`, `low`, `normal` (default) or `high`. Apple says to use `high` to "attempt to deliver the notification immediately".
- **`Topic`** (§5.4):
  - At most 32 URL-safe base64 characters.
  - A new message with the same topic **replaces** an undelivered one on that subscription. This is a protocol-level collapse key, relevant to batching and dedupe.
  - The topic is not encrypted.
- **Status codes:**
  - 201 means the push service *accepted* the message. It does not mean the message was delivered.
  - 429 means rate limiting and SHOULD include `Retry-After` (§8.4).

**Visible-notification rule:** Every push must result in a visible notification.
- Chrome and Edge reject `subscribe()` unless `userVisibleOnly: true`. ✅ (MDN)
- Apple: "Safari doesn't support invisible push notifications … If you don't, Safari revokes the push notification permission for your site." ✅
- WebKit says that for classic Web Push, a handler that doesn't show a notification gets its push subscription revoked. ✅ [WebKit: Meet Declarative Web Push](https://webkit.org/blog/16535/meet-declarative-web-push/)

**Declarative Web Push (Safari):** The payload is JSON.
- `{"web_push": 8030, "notification": {"title": …, "navigate": <url>, …}}`, where `title` and `navigate` are required.
- Optional fields include `body`, `lang`, `dir`, `silent` and `app_badge`.
- It is shown without running service-worker JS. A service worker may still modify the notification, and if it fails the declarative content is shown as a fallback.
- WebKit says it is backward compatible with classic Web Push.
- It is subscribed through `window.pushManager`.
- **Versions:** BCD lists 18.4 for both Safari and iOS. The WebKit post mentions iOS/iPadOS 18.4 and "macOS 15.5 beta". ⚠️ I did not determine exactly when it reached macOS Safari stable.
- ✅ (WebKit DWP post, BCD)

## 6. Permission-prompt rules

**User gesture:** The gesture requirement is enforced in these browsers:
- **Safari/iOS:** The request must come "in response to direct user interaction" (WebKit). Apple: "call the push subscription method immediately from the gesture's event handler code." ✅
- **Firefox 72+:** `requestPermission()` and `subscribe()` work only from a user gesture. ✅ (BCD notes)
- **Chrome:** In the sources I checked, a gesture is *not* hard-required. Gesture affects which UI appears instead. MDN calls a gesture "best practice" and says browsers are moving to disallow prompts without one. ✅ (MDN)

**Chrome prompt UI:**
- From Chrome 98, notification requests show as an address-bar chip. The full prompt bubble opens automatically only if the request followed a user gesture, or for "essential" permissions such as camera and mic. Without a gesture, the chip collapses to a blocked icon after a short delay. ✅ [Chrome: permission chip](https://developer.chrome.com/blog/permissions-chip)
- **Quieter UI (Chrome 80+):** Users who repeatedly deny notification prompts, and sites with very low acceptance rates, get a quieter prompt. Sites are unenrolled once acceptance improves. From Chrome 84, sites with "abusive" permission requests get it too, after a Search Console warning and a 30-day window. ⚠️ **secondary:** the Chromium post body did not load, so these details come from press coverage of it. [Chromium blog, 2020-01-07](https://blog.chromium.org/2020/01/introducing-quieter-permission-ui-for.html)
- **Auto-revocation (Chrome, announced 2025-10-10):** Chrome on Android and desktop removes notification permission from sites that have both very low engagement and high notification volume. **Installed web apps are excluded.** The user is told and can restore it. ✅ [Google blog: automatic notification permission](https://blog.google/chromium/automatic-notification-permission/)

**Cross-origin iframes:** Chrome and Firefox no longer allow notification permission requests from cross-origin `<iframe>`s. ✅ (MDN `requestPermission`)

**Permission states:** `requestPermission()` resolves to `granted`, `denied` or `default`. `default` behaves like denied. ✅ (MDN)

**iOS settings:** iOS manages the permission per web app in Settings → Notifications, and those notifications follow Focus modes. ✅ (WebKit 16.4)

## 7. Subscription expiry and invalidation signals

**What the spec says** ✅ (RFC 8030 §7.3, verified against raw text):
- "A push service MAY expire a subscription at any time."
- A push to an expired subscription "MUST return a **404 (Not Found)**".
- In the RFC, **410 (Gone)** is about *delivery receipts*: a message the service gave up on (§6.2–6.3). It is not about subscription expiry.

**What push services do in practice:**
- **Apple:** `410` = "The device token has expired". `404` = invalid `:path`, meaning a bad endpoint URL. ✅ (Apple doc)
- **web.dev:** For both 404 and 410, "the subscription has expired or is no longer valid" and should be removed from the database. ✅ [web.dev: Sending messages with web push libraries](https://web.dev/articles/sending-messages-with-web-push-libraries)
- **`web-push`:** Surfaces both through `WebPushError.statusCode`. The caller decides what to do. ✅ (source)

**Other errors to expect** ✅ (Apple doc):
- `400` with `reason` codes `BadTtl`, `BadUrgency`, `BadWebPushTopic` or `BadWebPushRequest`.
- `403` for VAPID or JWT problems.
- `413` payload too large.
- `429` too many requests to one device.
- `500` / `503` server errors.

**`expirationTime`:** May be `null`. Support: Chrome 60, Firefox 96, Safari 16/16.4. ✅ (MDN, BCD). ⚠️ I found no source saying how often browsers actually set a non-null value.

**`pushsubscriptionchange`:**
- MDN: fires for a change "triggered outside the application's control", which "may occur if the subscription was refreshed by the browser, but it may also happen if the subscription has been revoked or lost". MDN marks it not Baseline, and notes that a `fetch()` from the handler may fail if the device is offline. ✅
- Support: **Chrome only since 138**; **Firefox partial** (no `oldSubscription`/`newSubscription`); macOS Safari 16; **iOS Safari: not supported**. ✅ (BCD 8.1.9)
- **Fact:** A server cannot rely on this event to learn about invalidation on iOS or older Chrome. The send-time 404/410 response is the signal that works everywhere.

**Other ways a subscription ends:**
- The user revokes permission, or uninstalls the Home Screen app.
  - ⚠️ That uninstalling removes the subscription is inferred; I found no explicit source.
- Safari revokes the subscription after a push without a visible notification. ✅
- Chrome auto-revokes for low-engagement, high-volume sites (not for installed apps). ✅
- The VAPID key is rotated, which invalidates subscriptions bound to the old key. ✅ (RFC 8292 §4.2)

**Devices:** Each browser profile or installed app on each device has its own subscription and endpoint. One person may have several. ⚠️ (implied by the per-registration model; not explicitly verified)

## 8. Stack fit, factual only

- Sending is one outbound HTTPS POST per subscription per message, to whichever push service the endpoint names: FCM for Chrome, Mozilla autopush, or `*.push.apple.com`.
  - **Rate limits:** Apple caps unacknowledged requests at 100 per HTTP/1.1 connection, and HTTP/2 streams at `SETTINGS_MAX_CONCURRENT_STREAMS`. ✅
  - **Third-party services:** No account or third-party service is needed beyond the VAPID key pair. Apple needs no developer membership. ✅
- The data to store for each subscription is: `endpoint`, `p256dh`, `auth`, the VAPID key used, and the owning user. The key is needed because rotation invalidates subscriptions. ✅ (RFC 8292, MDN)

## Open questions / not verified

1. Whether push works on iOS 26 for a Home Screen app that has no manifest (BCD and WebKit 26 disagree).
2. The current EU status of Home Screen web apps.
3. The Android 13+ OS notification permission, and how it interacts with Chrome's site permission.
4. Whether Apple enforces its "≤ 1 JWT refresh per hour" rule against `web-push`'s re-sign-per-request behaviour.
5. Whether `@types/web-push@3.6.4` matches the `web-push@4` ESM API.
6. The exact thresholds for Chrome's quieter UI (primary post not readable).
7. How often `expirationTime` is non-null in practice.
