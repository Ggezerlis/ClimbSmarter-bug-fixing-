# ClimbSmarter.app: Bug Report (no-card subscription testing)

**Tested:** 2026-09-30, production (`https://climbsmarter.app`)
**Method:** Headless Chromium (Playwright), desktop 1280px and mobile 390px viewports, plus direct API calls made from the logged-in browser session. The client bundle was read to learn endpoints and validation rules.
**Confidence tags:** **(certain)** = reproduced with captured evidence; **(likely)** = strong inference; **(guessing)** = unverified.

## Headline

**The no-card change is incomplete.** Test mode is on (`testModeEnabled: true`), and Pro and Full activate with no card. But a live Stripe checkout can still be created, and most marketing copy still says users are charged today. Bugs 1 and 3 are the ones to fix before you show this to anyone.

## Summary

| # | Severity | Area | Bug |
|---|---|---|---|
| 0 | Medium (was High) | Post-onboarding | Black screen after onboarding: reported fixed by the owner; **residual:** subscription choices sometimes need a manual refresh to appear. Dashboard "Building Your Plan…" spinner also never auto-updated in my test |
| 1 | High | Subscription / Stripe | Live Stripe checkout session still created while test mode is on |
| 2 | High | Dashboard | Blank dashboard plus request storm for a verified user with no profile |
| 3 | Medium | Copy | Home, `/preview`, `/plan-ready` still say "charged today / immediately" |
| 4 | Medium | Naming | Tier names swapped between UI and API/errors (Pro vs Full) |
| 5 | Medium | Gating | Pro dashboard shows the Nutrition widget and Workout Timer that Pro doesn't include |
| 6 | Medium | Settings | Wrong billing text, dead "Manage Billing", cancel has no end date or undo |
| 7 | Medium | Auth | Unverified email can activate Free through the API; paid activation correctly blocks |
| 8 | Medium | Security | No throttling seen on login and check-email; check-email enumerates accounts |
| 9 | Low-Med | UI | Plan-card CTA text overflows on desktop; cards clipped on mobile |
| 10 | Low-Med | UI | `/preview` overflows horizontally on mobile; modal misplaced |
| 11 | Low | UX | Two competing Monthly/Yearly controls on `/subscribe` for existing subscribers |
| 12 | Low | Copy | Feature lists differ between home, `/subscribe` and `/preview` |
| 13 | Low | Onboarding | Silent defaults, weak DOB and injuries validation |
| 14 | Low | Copy | Pluralization, "link" vs "code", "expires in 10 hours" |
| 15 | Low | Perf/hygiene | Redundant polling, unused preload warning, missing security headers |
| 17 | Medium | Onboarding | `/subscribe` ignores the server-saved profile and sends you back to onboarding step 1 |
| 18 | Medium | Onboarding | Free user with no profile is bounced from `/onboarding` to a broken dashboard; the result flips between spinner and empty page |
| 19 | Medium | Subscribe page | Nav bar (sidebar / bottom tabs) shown to brand-new accounts that have no plan or dashboard yet |
| 16 | Low | Routing | Plan choice lost from home pricing; authed users can view `/login` |

Fix prompts for Replit, one per bug, are at the end of this file (section "Replit prompts").

---

## 0. Black screen after onboarding / stuck "Building Your Plan…": Medium (certain when tested; owner reports a fix since)

> **Status update (owner, after this test):** the black screen after onboarding is fixed. **Remaining issue:** right after onboarding, the choose-a-subscription page sometimes does not display its plan options until the user refreshes the page. It is intermittent and I have not reproduced it yet. It may be the same root cause as below: data (profile, plan or subscription state) is fetched once and never re-fetched when it becomes ready.
> The findings below are from my test before the fix and may no longer reproduce.

- **Repro (mobile 390px and desktop 1280px, same result):**
  1. Log in as a verified user, complete all 8 onboarding steps, press **Generate my plan** (Generating → `/preview`).
  2. Press **Subscribe**, then choose **Activate Pro test access**. You land on `/dashboard`.
- **Actual:**
  - The screen shows only a small spinner, "Building Your Plan… This page will update automatically", and two buttons on a near-black background. See the evidence screenshots.
  - It **never advances**. I watched for ~22s after activation and the page stayed there. The browser made only two `GET /api/training/plan` calls (at ~1s and ~3s after landing), and then **stopped polling**.
  - The plan does exist: the same call returned **200** right after, and a **fresh page load shows the full dashboard** immediately.
- **Why users call it a black screen:** the page is almost entirely dark, with no navigation or content. A user who doesn't reload assumes it is broken.
- **Fix:**
  - Keep polling `/api/training/plan` (for example every 2–3s, with a timeout and an error state), or invalidate the plan query when generation finishes.
  - Make **Check now** and **Redo onboarding** work as escape hatches. I could not reliably reach "Check now" in the stuck state, because the screen alternated between this spinner and an empty dashboard (see #18).
  - Show an error and a retry button if generation fails, instead of an endless spinner.
- **Evidence:** `bug-report-screenshots/10-dashboard-building-plan-stuck-mobile.png`, `11-dashboard-building-plan-stuck-desktop.png`.
- **Not reproduced:** a fully blank page immediately after **Generate**. Generating and `/preview` both rendered on mobile and desktop. If you see the black screen at a different step, or on a specific device or browser (for example Safari or an in-app browser), tell me which and I'll target it.

## 1. Live Stripe checkout is still reachable in test mode: High (certain)

- **Repro:** As a verified user with `testModeEnabled: true`, run `POST /api/stripe/create-checkout-session {"plan":"basic","billing":"monthly"}`.
- **Actual:** `200 {"url":"https://checkout.stripe.com/.../cs_live_..."}`. It is a **live-mode** session, not a test one.
- **Expected:** With no-card mode on, the endpoint should refuse (409/403) or the flag should not be bypassable.
- **Risk:** The UI hides this path, but anyone who calls the API can still be sent to a real payment page. That contradicts "subscriptions don't require a card", and it will silently take money if a user pays. I created one unpaid session while testing. It should expire.

## 2. Blank dashboard and request storm when there is no profile: High (certain)

- **Repro:**
  1. Click **Sign up** on the home page and register directly, without onboarding.
  2. Verify the email. You are correctly sent to `/onboarding`.
  3. Leave and open `/dashboard` directly.
- **Actual:** The main area stays empty and the header shows "Athlete / U" instead of the user's name. There is no redirect to onboarding, and no error is shown.
- **Measured:**
  - **91 API requests in 10s** on account C, and **~271 in 30s** on account B. That is `GET /api/auth/user` about 6/s and `GET /api/subscription` about 3/s, plus repeated `POST /api/training/plan/generate`, which returns `{"status":"no_profile"}`.
  - `/subscribe` handles the same state correctly by redirecting to `/onboarding`. `/dashboard` doesn't.
- **Fix:**
  - Redirect to `/onboarding` when `/api/profile` returns 404 or `generate` returns `no_profile`.
  - Stop the refetch loop (likely a query that invalidates itself on every render or has a very short refetch interval).
- **Evidence:** `bug-report-screenshots/07-blank-dashboard-no-profile.png`

## 3. Copy still says users are charged: Medium (certain)

The `/subscribe` and Settings pages are correct for test mode. Other pages are not:

- **Home pricing:** "Paid subscriptions are charged immediately." and "€11.99/month charged today" / "€38.99/month charged today".
- **`/preview` modal:** "Paid plans charge today · Free requires no card".
- **`/plan-ready`:** "Paid subscriptions are charged today", "Paid plans charge today, Clear, immediate billing". See `09-plan-ready-charge-copy.png`.
- **Effect:** Users are told they'll be charged, then get free access. That looks like a billing bug, or worse, like a dark pattern.

## 4. Pro and Full names are swapped between UI and backend: Medium (certain)

- **Internal plan ids:** `basic` is shown as "Pro" and `pro` is shown as "Full".
- **Errors that leak the wrong name:**
  - `/api/nutrition/goals` for the Pro tier: `403 "Pro subscription required"`. In the UI, Nutrition is a Full feature.
  - `/api/coach/chat`: "Upgrade to Pro to use AI Coach chat."
  - `/coach` page: the heading says "Upgrade to Full", but the footer says "AI Coach is a Pro feature".
  - Free user on `/progress`: "…with a Pro membership."
  - `/api/coach/status` returns `isPro:true` only for the Full plan.
- **Effect:** Users are told to "upgrade to Pro" when they already have Pro. It will also confuse support and the future Stripe price mapping.

## 5. Pro sees Full-only UI: Medium (certain for the widget, guessing for the timer)

- On Pro, the dashboard shows a **"Nutrition Today"** widget (0 kcal / 0 g / 0 g / 0 L) while `/api/nutrition/*` returns 403. The `/nutrition` page itself is correctly locked. See `06-pro-dashboard-nutrition-widget.png`.
- The Pro card lists "Workout Timer ✗", yet the Pro dashboard shows a **Workout Timer** row. I did not open it, so whether it works on Pro is untested. (guessing)

## 6. Settings and cancel flow assume real billing: Medium (certain)

- **Wrong charge text:**
  - Test Pro shows "Active · billed €11.99/month".
  - Test Full shows "billed €38.99/month".
  - Nothing is billed. See `04-settings-billing-copy.png`.
- **Dead Manage Billing:** Clicking it gives the error "No billing account found. Please subscribe first." to a user who is subscribed. The row promises "Update payment method, view invoices, cancel via Stripe".
- **Cancel dialog:** It says "keep full access until the end of your current billing period", but a test plan has no period. After cancelling:
  - `cancelAtPeriodEnd: true`, but no end date is shown anywhere.
  - Settings says "Access continues until end of billing period".
  - The **Cancel Subscription** button is still shown, and there is no Resume or Reactivate.
  - `/subscribe` still labels the cancelled plan **"CURRENT PLAN"** (a disabled button).
- **Open question:** How and when does a cancelled test plan actually end? It is undefined.

## 7. Email verification is not enforced on the Free path: Medium (certain)

- **Repro:** With an unverified account, `POST /api/subscription/activate-free` returns **200** and `plan:"free", hasAccess:true`.
- **Compare:** `POST /api/subscription/activate-test` correctly returns `403 "Verify your email before selecting a plan."`
- The UI redirects unverified users to `/verify-email-sent`, so this is API-only. But the rule should be enforced on the server for every plan endpoint.

## 8. No throttling seen; account enumeration: Medium (likely)

- 15 consecutive failed logins and 25 `check-email` calls all succeeded with no `429`. I stopped there on purpose, so a higher threshold may exist. (guessing)
- `POST /api/auth/check-email` returns `{"exists":true,"verified":true}` for any email. Combined with no throttling, that lets anyone list registered emails and see which are verified. It is normal for this "enter email first" UX, but it should be rate limited.
- `forgot-password` returns a generic success. That is good.

## 9. Plan-card layout: Low-Medium (certain)

- **Desktop (1280px):**
  - CTA labels overflow their buttons: "ACTIVATE PRO TEST ACCESS", "SWITCH TO FULL TEST ACCESS" (clipped to "TCH TO FULL TEST ACCE…"), and "AVAILABLE ON FREE SIGNUP".
  - The three CTAs are not bottom-aligned across cards.
- **Mobile (390px):** The cards run past the right edge. The right border is missing on Free and Pro, the "RECOMMENDED" badge is cut off, and the buttons are clipped. `scrollWidth` reports no overflow, so the horizontal scroll is hidden (`overflow-x: hidden`).
- **Evidence:** `01-subscribe-desktop-clipped-cta.png`, `02-subscribe-mobile-cards-clipped.png`.

## 10. `/preview` is broken on mobile: Low-Medium (certain)

- `document.scrollWidth > innerWidth` at 375px, so the page scrolls horizontally.
- The "Unlock Your Full Plan" modal is offset to the right and overlaps the day list. See `03-preview-mobile-overflow.png`.

## 11. Two billing-cycle controls: Low (certain)

Existing subscribers see a "Billing Cycle" panel (with a confirm button) and a separate Monthly/Yearly toggle above the cards. Clicking "Yearly" in the panel does not change the card prices. Two controls doing related things is confusing. See `05-subscribe-two-billing-controls.png`.

## 12. Feature lists disagree across pages: Low (certain)

- **Home Pro:** Says "Daily guided workouts" and "Streak tracking". `/subscribe` says Pro has no Workout Timer and does not mention streaks.
- **Home Full:** Says "Unlimited AI Coach access", "Advanced progress analytics" and "Streak discount milestones". `/subscribe` lists none of these.
- **Early AI Access:** Appears only in the "What you unlock with Full" panel, not on the Full card.
- **Home page:** Omits the Free tier entirely, although the app has one.
- **Also:** `/preview` and `/plan-ready` are publicly reachable with generic content ("matched to your grade") for someone who never did onboarding. (likely by design as a funnel demo)

## 13. Onboarding defaults and validation: Low (certain)

- **Preselected answers:**
  - Grade **V5** is preselected on step 2 (`08-onboarding-grade-preselected.png`). A user can click Continue without answering.
  - Equipment: I never selected any, but the payload sent `"equipment":["fingerboard"]`. That is a hidden default, and the generated plan included **Max Hangs**.
- **DOB:** The form shows messages for future dates and under-15, which is good. But **1850-01-01** is accepted silently, and the input's `min=1926` is not enforced when typed.
- **Injuries field:** The textarea has no `maxlength`. 5,026 characters were accepted client-side. Server-side limits were not tested.
- **Days:** The last remaining day can't be deselected. That is sensible, but there is no message explaining why.

## 14. Copy and text bugs: Low (certain unless noted)

- **Pluralization:** "Set a start time for each of your **1 training day**".
- **Wrong wording:** The gate toast says "Check your inbox for the verification link", but the flow uses a **6-digit code**.
- **Code expiry:** The email says "It expires in **10 hours**". This is probably meant to be minutes. (guessing)
- **Placeholder:** Visiting `/verify-email-sent` or `/verify-email` while logged out shows the placeholder "your email address" instead of redirecting.
- **Weak signup error:** A weak or invalid signup body returns the bare API message "Invalid request body". The client shows it unchanged, with no explanation of the password or email rule. (likely)

## 15. Performance and hygiene: Low

- **Redundant polling:**
  - `GET /api/auth/user` is called ~4 times per onboarding step.
  - `GET /api/subscription` is called ~5 times when loading `/subscribe`.
- **Preload warning:** On every non-home route, Chrome warns that `hero-bg.webp` was preloaded but not used. This wastes bandwidth on the login, signup and legal pages. (certain)
- **Malformed JSON:** Returns a raw Express HTML "Bad Request" page instead of JSON. (certain)
- **Headers:**
  - No CSP, `X-Frame-Options` or `Referrer-Policy` on the HTML document (`X-Frame-Options: DENY` exists only on API responses).
  - `X-Powered-By: Express` is exposed.
  - Two different HSTS headers are sent on API responses.
- **Plan-ready screen:** After the plan existed (verified via API at ~17s), the dashboard still showed "Building Your Plan… This page will update automatically" 3s later. A fresh load showed it correctly. I did not measure the auto-update delay. (guessing)
- **Nutrition targets:** Full-plan defaults were an identical **2500 kcal / 160 g / 300 g / 2.5 L** for a user with no body data. The "AI macro targets" claim looks unpersonalized. (guessing)

## 17. `/subscribe` redirects to onboarding even though the profile is saved: Medium (certain)

- **Repro:** Complete onboarding so the profile is saved (`GET /api/profile` returns it), then open `/subscribe` in a new browser session or after clearing site data, without a plan yet.
- **Actual:** You are sent to `/onboarding` step 1 ("One more step: complete your profile"). The code checks the browser-local onboarding data and the plan state, not the server profile.
- **Effect:** Users who onboard on one device or browser, then open the site on another, must redo all 8 steps before they can pick a plan.

## 18. Free user without a profile hits a dead end: Medium (certain)

- **Repro:** Account with `plan: free` and no profile (reachable via `activate-free`, and possibly after cancelling and clearing data): open `/onboarding`.
- **Actual:** You are redirected to `/dashboard`. Depending on timing, the dashboard shows either the full-screen "Building Your Plan…" spinner (no sidebar) or an empty page with only the sidebar. The request storm from bug #2 runs in the background.
- **Effect:** The user cannot complete onboarding, and the page gives no error or working way out.

## 19. Navigation bar is shown on the choose-a-subscription page for brand-new accounts: Medium (certain)

- **Repro:**
  1. Create an account and verify the email (or finish onboarding), so the account has no plan yet.
  2. Land on `/subscribe`.
- **Actual:** The full app navigation is shown: the sidebar on desktop (Dashboard, AI Coach, Weekly Check-in, Nutrition, Subscription, Settings, plus the "Get the app" box and the sign-out button), and the bottom tab bar on mobile. See `01-subscribe-desktop-clipped-cta.png` and `02-subscribe-mobile-cards-clipped.png`, both taken on a new account with `plan: none`.
- **Expected:** A user who has just signed up has no plan or dashboard yet, so this page should be a focused, nav-free screen (logo and, at most, a sign-out link). Every nav item leads to a page that is empty, locked or a dead end.
- **Effect:** Users click "Dashboard" and land on a dashboard that does not exist for them yet. I observed exactly those broken states for accounts with no plan or profile (see #2 and #18). The nav also pulls attention away from the plan choice.
- **Fix:** Render `/subscribe` without the app layout while the user has no plan (`plan: none`). Restore the nav once a plan, including Free, is active.
- **Related:** The same applies to any other page shown before a plan exists (for example while onboarding is incomplete).

## 16. Routing: Low (certain)

- Home "SUBSCRIBE" (Pro/Full) and "TRY IT FREE" both go to `/onboarding`. The chosen plan and billing cycle are not carried through.
- An already-authenticated user can still open `/login`. It does not redirect to the dashboard.
- `/pricing` is a 404. The home "VIEW PRICING" button scrolls to an anchor, so this only matters for external links.

---

## Checked and OK

- **Registration and verification:** Password mismatch error; submit disabled until terms are ticked; code entry works with typing and paste, rejects wrong codes (400) and letters.
- **Server validation:** `activate-test` rejects invalid plan, billing, missing fields and array input (400). Login errors are generic. CORS does not reflect foreign origins. `returnTo=https://evil…` on login is **not** an open redirect.
- **Access control:** All private routes redirect to `/enter` when logged out. Nutrition and AI Coach are locked for Free and Pro and unlocked for Full (except the UI leaks in bug 5).
- **Plan generation:** Completes in about 17s after activation. Grade conversion table (V0–V17 to Font) is correct. The age limit (15) matches the Terms and Privacy Policy.

## Not tested

- Real card payment and Stripe webhooks; the live checkout session I created was never opened.
- Google and Apple sign-in, the iOS and Android apps, and email delivery beyond the verification code.
- Password reset end to end, Settings "Edit" training profile, workout completion and timer, and the check-in submit.
- Real devices and other browsers. Only Chromium with emulated viewports.

## Test data left on production

Three accounts were created with aliases of your address and are still active:

| Account | State |
|---|---|
| `george+cstest0930@gezerlis.gr` | Pro, yearly, test subscription; has a generated plan |
| `george+cstest0930b@gezerlis.gr` | Free, no profile (via API) |
| `george+cstest0930c@gezerlis.gr` | Verified, no profile |

Password for all three: shared in the chat session, deliberately not stored here. The three verification emails are in your Gmail inbox. There is one unpaid live Stripe checkout session. There is an account-delete endpoint (`/api/auth/account`), which I did not call.

---

# Replit prompts (one per bug)

**How to use:** paste the **Shared rules** block first (or at the top of every prompt), then paste one bug prompt per Replit run. Do them one at a time and verify each on the live site before starting the next. Suggested order: 1, 3, 2, 18, 17, 0, 19, 7, 4, 5, 6, 8, then the rest.

Each prompt has: **Context** (what is wrong), **Task**, **Recommended fix**, **Must NOT do**, **Done when**.

## Shared rules (prepend to every prompt)

```text
You are fixing one specific bug in ClimbSmarter.app (React SPA with wouter routing and react-query, Express backend, Stripe, Postgres). Read the whole prompt before touching code.

Ground rules:
- Fix ONLY the bug described. Do not refactor, rename, restyle, upgrade dependencies, or "clean up" unrelated code.
- Before editing, find the relevant code and tell me the file paths and the root cause you found. If the root cause differs from what I describe, say so and fix the real cause.
- Keep the app's current "test mode" behavior: paid plans (Pro, Full) activate WITHOUT a card, and Free needs no card. Do not remove test mode, do not change the testModeEnabled flag, and do not change how activate-test / activate-free succeed for valid users.
- Naming: internal plan id `basic` = UI name "Pro" (EUR 11.99/month), internal `pro` = UI name "Full" (EUR 38.99/month). Do not rename internal ids or migrate stored data unless this prompt explicitly says so.
- Do not touch Stripe live keys, webhooks, price ids, or the database schema unless this prompt explicitly says so. Never run destructive DB commands. Never delete users, subscriptions or plans.
- Do not add new dependencies unless unavoidable; if you must, name them and why.
- Do not disable, skip or delete tests or validation to make something pass. Do not hide errors with empty try/catch.
- Do not change unrelated copy, pricing, or layout.
- After the change, verify in the running app (desktop 1280px AND mobile 390px where UI is involved) and paste the exact steps you ran and their results. If you could not verify something, say so plainly instead of claiming it works.
- End with: files changed, what changed in one sentence each, how to test, and anything you noticed but deliberately did NOT change.
```

## Prompt 0: Post-onboarding black screen / stuck "Building Your Plan…" and subscription choices needing a refresh

```text
Context: After onboarding, users get a near-black screen with a spinner ("Building Your Plan… This page will update automatically"). The black screen itself was reportedly fixed, but two problems remain: (1) right after onboarding the choose-a-subscription page (/subscribe) sometimes does not show the plan options until the user refreshes; (2) in my testing, after activating Pro the dashboard stayed on "Building Your Plan…" forever. The browser made only two GET /api/training/plan calls (about 1s and 3s after landing) and then stopped polling, although the plan existed (the same call returned 200 shortly after and a fresh page load showed the full dashboard). Likely root cause: data (profile, subscription, plan) is fetched once and never refetched or invalidated when it becomes ready, or the polling stops after a fixed number of attempts.

Task: Make the post-onboarding flow self-updating so that no manual refresh is ever needed, and never leave the user on an endless spinner.

Recommended fix:
1. Find the query that loads /api/training/plan for the dashboard. While the plan is not ready (status generating/pending or 404), poll every 2-3 seconds with react-query refetchInterval that returns false only when the plan is ready or an error/timeout state is reached. Add a timeout (for example 90 seconds) after which an error state with a working "Try again" button is shown.
2. When onboarding completes and when a subscription is activated, invalidate the react-query keys for the user, subscription, profile and training plan so /subscribe and /dashboard always render fresh data.
3. On /subscribe, do not gate rendering of the plan cards on a query that may still hold stale "no profile / no subscription" data from before onboarding finished. Show a loading skeleton while the queries are in flight, then the cards. Use refetchOnMount: "always" for the subscription and profile queries on this route.
4. Make the "Check now" and "Redo onboarding" buttons on the building screen actually work (Check now = refetch the plan immediately).
5. If plan generation fails (server returns an error), show an error message and a retry button instead of the spinner.
6. Make the loading screen use the normal app background and include the app header/logo so it never looks like a black screen.

Must NOT do: do not add window.location.reload() or a forced page refresh as the fix; do not poll faster than every 2 seconds; do not poll forever without a timeout; do not change the plan generation algorithm; do not change the /api/training/plan response shape; do not change subscription pricing or activation logic.

Done when: (a) finishing onboarding then landing on /subscribe shows the plan cards every time without a refresh (test at least 10 times, including throttled network); (b) after activating a plan the dashboard switches from "Building Your Plan…" to the plan automatically within a few seconds of the plan existing; (c) a simulated generation failure shows an error and a working retry; (d) the network tab shows polling stopping once the plan is ready.
```

## Prompt 1: Live Stripe checkout still created in test mode

```text
Context: With testModeEnabled true, POST /api/stripe/create-checkout-session {"plan":"basic","billing":"monthly"} by a verified user still returns 200 with a LIVE Stripe checkout URL (cs_live_...). The UI hides this path, but anyone calling the API can be sent to a real payment page. That contradicts "subscriptions do not require a card".

Task: Make the backend refuse to create real Stripe checkout sessions while test mode is on.

Recommended fix:
1. In the create-checkout-session handler, at the very top after authentication, check the same test-mode condition the activate-test endpoint uses (reuse the existing helper or flag; do not duplicate the logic in a new way). If test mode is on, respond 409 with JSON {"error":"Card payments are disabled during test mode. Use /api/subscription/activate-test."} and do NOT call the Stripe API.
2. Apply the same guard to the other endpoints that create real billing objects while in test mode: /api/stripe/change-plan, /api/stripe/change-tier, /api/stripe/create-portal-session, /api/stripe/restore and /api/stripe/activate-checkout. For each one, return a clear 409 in test mode, unless it is needed to support existing real subscribers (in that case explain in your report and keep it working for them only).
3. Keep Stripe webhook handling unchanged so any pre-existing real subscriptions still sync.
4. Add a server log line (not user data) when a checkout attempt is blocked.
5. Add a test that proves the endpoint returns 409 in test mode and never calls the Stripe client (mock it).

Must NOT do: do not delete the Stripe integration or code; do not change or remove Stripe keys; do not touch the webhook endpoint; do not turn test mode off; do not make the guard depend on a client-sent parameter; do not return 200 with a fake URL.

Done when: calling create-checkout-session (and the other listed endpoints) as a verified user in test mode returns 409 and no Stripe session is created; activate-test and activate-free still work; turning test mode off restores the original behavior.
```

## Prompt 2: Blank dashboard and request storm for a verified user with no profile

```text
Context: A verified user who registered but never finished onboarding (no profile) who opens /dashboard directly sees an empty main area and "Athlete / U" in the header, with no redirect and no error. The browser also fires about 91 API requests in 10 seconds: GET /api/auth/user about 6/s, GET /api/subscription about 3/s, and repeated POST /api/training/plan/generate which returns {"status":"no_profile"}. /subscribe already handles this state correctly by redirecting to /onboarding.

Task: Redirect users without a profile to onboarding and stop the request loop.

Recommended fix:
1. In the dashboard route guard (or the dashboard component), when GET /api/profile returns 404 or /api/training/plan/generate returns status "no_profile", navigate (replace, not push) to /onboarding.
2. Find why the queries loop. Typical causes: a query key that changes every render, invalidateQueries called inside render or in an effect without a stable dependency array, refetchInterval that is too short, or an effect that calls generate on every state change. Fix the actual cause. After the fix, /api/auth/user and /api/subscription should each be requested once on load and then only on window focus or explicit invalidation.
3. Only call POST /api/training/plan/generate when a profile exists and a plan is missing, and never more than once per attempt (guard with a mutation state, not an effect that re-fires).
4. Handle the "no_profile" response as a normal state, not an error to retry.
5. Fix the header so it shows the user's real name (from /api/auth/user) or a sensible fallback, not "Athlete / U", when the profile is missing.

Must NOT do: do not silence the loop by disabling react-query globally; do not set staleTime to infinity everywhere; do not add setTimeout hacks; do not change what /api/profile or generate return; do not auto-create an empty profile on the server just to make the dashboard render.

Done when: opening /dashboard as a verified user with no profile redirects to /onboarding; the network tab shows no more than a handful of requests in the first 10 seconds (I expect under 10); no repeated generate calls; normal users with a profile see no change.
```

## Prompt 3: Copy still says users are charged

```text
Context: Test mode makes Pro and Full free (no card), but several pages still say users are charged: Home pricing ("Paid subscriptions are charged immediately." and "EUR 11.99/month charged today" / "EUR 38.99/month charged today"); the /preview modal ("Paid plans charge today · Free requires no card"); /plan-ready ("Paid subscriptions are charged today", "Paid plans charge today, Clear, immediate billing"). /subscribe and Settings already use correct test-mode copy.

Task: Make every user-facing string consistent with test mode, without lying and without breaking the future switch back to paid.

Recommended fix:
1. Search the whole client for strings containing: "charged", "charge today", "billed", "immediately", "billing", "card", "per month" in pricing contexts, and list every hit with file and line before editing.
2. Add ONE central source (for example a small config/util reading testModeEnabled from the existing subscription/config endpoint) and use it in all pricing copy, so the copy switches automatically between test mode ("Free during early access, no card needed") and paid mode ("Charged today"). Reuse the exact wording already used on /subscribe for test mode.
3. Update Home pricing, /preview modal and /plan-ready to use it. Keep the displayed EUR prices only if you label them clearly (for example "EUR 11.99/month after early access") and only if that is true; if unsure, ask me instead of inventing terms.
4. Check the email templates and the Terms/Privacy pages for the same wording and report (do not silently rewrite legal text).

Must NOT do: do not invent a promise (for example a specific free period or end date) that I did not give you; do not remove the pricing cards; do not hardcode test-mode copy in a way that cannot be reverted; do not change the Terms or Privacy text without telling me.

Done when: no page says the user will be charged today while test mode is on (I will grep the live pages), and flipping the test-mode flag off restores the "charged" wording automatically.
```

## Prompt 4: Pro and Full names swapped between UI and backend

```text
Context: Internal plan id basic is shown as "Pro" and internal pro is shown as "Full". Error messages leak the wrong name: /api/nutrition/goals for a Pro-tier user returns 403 "Pro subscription required" although Nutrition is a Full feature; /api/coach/chat says "Upgrade to Pro to use AI Coach chat."; the /coach page heading says "Upgrade to Full" but its footer says "AI Coach is a Pro feature"; a Free user on /progress sees "...with a Pro membership."; /api/coach/status returns isPro:true only for the Full plan.

Task: Make every user-visible name match the UI naming (basic = Pro, pro = Full) and stop internal names leaking.

Recommended fix:
1. Create one shared mapping (plan id -> display name, price, feature list) on the server and expose it to the client, or a shared constants file used by both. All messages must use the display name from that mapping.
2. Rewrite each message so it names the tier that actually unlocks the feature: Nutrition and AI Coach are Full features; say "Upgrade to Full". Fix each string listed above, then grep the server and client for other messages containing "Pro" or "pro subscription" and fix the ones that are wrong.
3. Do NOT rename the API field isPro or the internal ids (that would break the mobile apps and stored data). Instead add a new correctly named field, for example isFull or tier: "full", keep isPro unchanged for backward compatibility, and add a code comment explaining the mismatch. Tell me which clients read isPro.
4. Make sure the plan names in Settings, /subscribe, emails and error toasts agree.

Must NOT do: do not rename or migrate the stored plan values; do not remove isPro; do not change prices or which features belong to which tier; do not change Stripe price mapping.

Done when: every error and upgrade message for Nutrition and AI Coach says Full; a Pro user is never told to "upgrade to Pro"; API responses for existing fields are byte-for-byte compatible except for the new additive field.
```

## Prompt 5: Pro dashboard shows Full-only widgets

```text
Context: On the Pro tier the dashboard shows a "Nutrition Today" widget (0 kcal / 0 g / 0 g / 0 L) while /api/nutrition/* returns 403 for Pro. The /nutrition page itself is correctly locked. The Pro plan card also lists "Workout Timer" as not included, yet the Pro dashboard shows a Workout Timer row; I have not verified whether it works on Pro.

Task: Show each tier only the dashboard widgets it includes.

Recommended fix:
1. Locate the Nutrition Today widget and the Workout Timer row. Gate them with the SAME entitlement check the /nutrition page and the plan cards use (single source of truth, not a new ad-hoc condition).
2. For tiers without access, either hide the widget or render a clear locked state ("Nutrition is a Full feature" with an Upgrade button to /subscribe). Prefer a compact locked card so users can discover the upgrade.
3. Decide the Workout Timer rule from the plan card as the source of truth (Pro = no timer). If you find it conflicts with the home page copy ("Daily guided workouts"), do not guess: list both and ask me.
4. Do not fire the nutrition API calls at all for tiers without access (avoid pointless 403s).

Must NOT do: do not grant Pro access to make the widget work; do not change the server 403; do not change Full or Free behavior; do not change plan card contents in this task.

Done when: a Pro account shows no nutrition widget data and makes no /api/nutrition calls; Full is unchanged; a Free account is consistent with its own plan card.
```

## Prompt 6: Settings and cancel flow assume real billing

```text
Context: Settings for a test subscription says "Active · billed EUR 11.99/month" (Pro) or "billed EUR 38.99/month" (Full) although nothing is billed. "Manage Billing" errors with "No billing account found. Please subscribe first." for a subscribed user. The cancel dialog promises access "until the end of your current billing period", but a test plan has no period. After cancelling: cancelAtPeriodEnd is true, no end date is shown, Settings says "Access continues until end of billing period", the Cancel button is still shown with no Resume option, and /subscribe still labels the cancelled plan "CURRENT PLAN".

Task: Make the Settings billing section and cancel flow correct for test subscriptions (isTestSubscription).

Recommended fix:
1. For test subscriptions: show "Pro (test access)" / "Full (test access) - no card, not billed". Remove the price line or label it "Price after test period: ..." only if such terms exist (ask me otherwise).
2. Hide or replace "Manage Billing" for test subscriptions with a plain explanation ("You are on free test access, there is no billing account"). Do not call the Stripe portal endpoint for test users.
3. Define and implement cancel semantics for test plans and tell me what you chose. Recommended: cancelling a test plan takes effect immediately and returns the user to Free (simple, honest). If instead you keep access until a date, store and display that end date.
4. If a cancelled plan still has access, replace the Cancel button with "Resume subscription" (calls an existing/new reactivation path) and show the end date.
5. On /subscribe, label a cancelled-but-still-active plan "ENDS <date>" or "CANCELLED", not "CURRENT PLAN".
6. Update the cancel dialog text to match the chosen behavior.

Must NOT do: do not change how real (Stripe) subscriptions are cancelled or billed; do not delete the user's plan data or training plan when cancelling; do not call Stripe for test subscriptions; do not silently downgrade without telling the user.

Done when: a test Pro and a test Full user see correct text, no dead buttons, a defined end state after cancelling, and a way back (resume or resubscribe) that works.
```

## Prompt 7: Email verification not enforced on the Free path

```text
Context: POST /api/subscription/activate-free returns 200 (plan free, hasAccess true) for an account whose email is NOT verified. POST /api/subscription/activate-test correctly returns 403 "Verify your email before selecting a plan." The UI redirects unverified users so this is API-only, but the server must enforce the rule.

Task: Enforce email verification consistently on every plan-selection endpoint.

Recommended fix:
1. Extract the check used by activate-test into a small shared middleware or helper (for example requireVerifiedEmail) and apply it to activate-free, activate-test, and any other endpoint that grants access or creates a subscription (including the Stripe checkout/change endpoints and training plan generation if it grants plan access).
2. Return the same status and message as activate-test (403, "Verify your email before selecting a plan.") so the client already handles it.
3. Confirm the client shows the verification screen if it ever receives this 403.
4. Add tests: unverified user gets 403 on each endpoint; verified user is unchanged.

Must NOT do: do not weaken the check on activate-test; do not auto-verify emails; do not block login itself; do not touch the verification code flow.

Done when: an unverified account cannot obtain any plan (including Free) through any API call, and verified accounts behave exactly as before.
```

## Prompt 8: No throttling on login/check-email; account enumeration

```text
Context: 15 consecutive failed logins and 25 POST /api/auth/check-email calls all succeeded with no 429. check-email returns {"exists":true,"verified":true} for any email, which allows listing registered emails and their verification state. forgot-password already returns a generic response (good).

Task: Add rate limiting and reduce enumeration, without breaking the "enter your email first" UX.

Recommended fix:
1. Add express-rate-limit (or an equivalent already in the project) with a store that works on this deployment. Limits (tune if needed and tell me): login 5 failed attempts per 15 minutes per IP+email pair; check-email 10 per minute per IP; register, resend-verification, verify-email and forgot-password 5 per 10 minutes per IP and per email. Return 429 with a Retry-After header and a friendly JSON message; make the client show it.
2. Make sure the limiter uses the correct client IP behind the proxy (set trust proxy correctly), not the proxy's IP, otherwise all users share one bucket.
3. For check-email: keep the behavior needed by the UI, but stop returning the "verified" flag to unauthenticated callers if the UI can route without it (for example, send unverified users through login and show the verification step afterward). If that is not feasible, keep it and rely on the rate limit; explain the trade-off to me.
4. Add a small delay or uniform response time on failed login so timing does not reveal whether an email exists.

Must NOT do: do not lock accounts permanently; do not block by IP alone in a way that penalizes shared networks (schools, gyms); do not log passwords or full emails in plain text; do not change password hashing; do not break Google/Apple sign-in.

Done when: the 6th failed login in the window returns 429 with Retry-After; normal logins still work; the client shows a readable message on 429; the limiter counts real client IPs.
```

## Prompt 9: Plan-card CTA overflow and clipped cards on mobile

```text
Context: On /subscribe at 1280px the CTA labels overflow their buttons ("ACTIVATE PRO TEST ACCESS", "SWITCH TO FULL TEST ACCESS" clipped to "TCH TO FULL TEST ACCE...", "AVAILABLE ON FREE SIGNUP") and the three CTAs are not bottom-aligned across cards. At 390px the cards run past the right edge: the right border is missing on Free and Pro, the RECOMMENDED badge is cut off and buttons are clipped. document scrollWidth reports no overflow because overflow-x is hidden somewhere, which is hiding the real bug.

Task: Fix the plan card layout on desktop and mobile.

Recommended fix:
1. Find the element that has overflow-x hidden and the child that is wider than the viewport (use browser devtools; look for fixed widths, min-w values, negative margins, or grid columns with min-content). Fix the child, not by adding more overflow hidden.
2. Make each card a flex column with the CTA pushed to the bottom (mt-auto) so buttons align across cards. Let CTA text wrap to two lines (whitespace-normal, text-center, leading-tight) or shorten the labels ("Start Pro", "Switch to Full") and keep the details in a subtitle.
3. On mobile, stack the cards in one column with the page's normal side padding; ensure the RECOMMENDED badge is fully visible.
4. Test at 320, 375, 390, 768, 1024 and 1280px.

Must NOT do: do not change plan names, prices, features or which plan is recommended; do not shrink the font below 14px to force a fit; do not use transform scale hacks; do not remove overflow hidden globally without checking every page.

Done when: no clipped text or borders at any listed width, CTAs are aligned, and document.documentElement.scrollWidth <= window.innerWidth at every width.
```

## Prompt 10: /preview mobile overflow and misplaced modal

```text
Context: At 375px, /preview scrolls horizontally (scrollWidth > innerWidth), and the "Unlock Your Full Plan" modal is offset to the right and overlaps the day list.

Task: Fix horizontal overflow and center the modal on mobile.

Recommended fix:
1. Identify the overflowing element (devtools: outline every element with getBoundingClientRect().right > innerWidth) and fix its width (max-w-full, min-w-0 on flex/grid children, break-words on long text).
2. Make the modal a proper centered dialog: fixed inset-0, flex items-center justify-center, max-w with side margins (for example calc(100vw - 32px)), max-h 90vh with internal scroll, and a backdrop. Lock body scroll while it is open.
3. Make sure the modal never covers essential controls without a way to close it, and that its CTA and close button are reachable on 320px screens.
4. Test at 320, 375, 390 and 768px.

Must NOT do: do not hide the horizontal scroll with overflow-x hidden as the only fix; do not change the modal copy in this task (copy is bug 3); do not change the preview plan content.

Done when: no horizontal scroll at 320-768px, the modal is centered and fully visible, and closing it works.
```

## Prompt 11: Two competing billing-cycle controls

```text
Context: Existing subscribers see a "Billing Cycle" panel with a confirm button AND a separate Monthly/Yearly toggle above the plan cards on /subscribe. Choosing Yearly in the panel does not change the card prices. Two controls with related effects are confusing.

Task: Keep one billing-cycle control that drives both price display and the change action.

Recommended fix:
1. Keep the toggle above the cards as the single control. Its value drives the displayed prices AND the billing value sent when the user changes plan.
2. Remove the separate Billing Cycle panel, or turn it into a read-only line ("Current billing: Yearly") and a single "Apply" action only when the selected cycle differs from the current one.
3. In test mode there is no real billing, so make sure switching cycle for a test subscription only updates the stored test cycle (existing activate-test with the new billing) and never touches Stripe.
4. Keep both states in one piece of React state to avoid drift.

Must NOT do: do not change prices; do not call Stripe in test mode; do not remove the ability for real subscribers to change cycle; do not change plan-switch behavior beyond the control consolidation.

Done when: exactly one cycle control exists, the cards' prices follow it, and applying a cycle change updates the account correctly.
```

## Prompt 12: Feature lists disagree across pages

```text
Context: Feature lists differ between Home, /subscribe and /preview. Home Pro says "Daily guided workouts" and "Streak tracking" while /subscribe says Pro has no Workout Timer and does not mention streaks. Home Full mentions "Unlimited AI Coach access", "Advanced progress analytics" and "Streak discount milestones" that /subscribe does not list. "Early AI Access" appears only in the "What you unlock with Full" panel, not on the Full card. The Home page omits the Free tier entirely.

Task: Create one source of truth for plan features and render every pricing surface from it.

Recommended fix:
1. Create a single constants module (name, internal id, price, billing, feature list with included/not included per tier) and use it on Home, /subscribe, /preview, /plan-ready and the "What you unlock" panel.
2. BEFORE writing, list every feature claim currently shown per page and per tier, and ask me to confirm which are true. Do NOT decide what a tier includes yourself. Use the actual entitlement checks in the code (what the server really allows) as evidence, and flag any claim that the code does not support.
3. Add the Free tier to Home pricing.

Must NOT do: do not add or remove features from a tier on your own; do not change server entitlements; do not invent feature claims; do not change prices.

Done when: I confirmed the feature matrix, every page shows the same matrix, and any claim not backed by code is listed for me to decide.
```

## Prompt 13: Onboarding defaults and validation

```text
Context: (a) Grade V5 is preselected on onboarding step 2, so a user can continue without answering. (b) Equipment: without selecting anything, the payload sent "equipment":["fingerboard"] and the generated plan included Max Hangs. (c) DOB 1850-01-01 is accepted silently and the input's min=1926 is not enforced when typed. (d) The injuries textarea has no maxlength (5,026 characters accepted client-side); server limits untested. (e) The last remaining training day cannot be deselected and there is no message explaining why.

Task: Fix the defaults and validation on client AND server.

Recommended fix:
1. Grade: no preselection; require an explicit choice before Continue is enabled.
2. Equipment: do not send a hidden default. If the user selects nothing, either require at least one option (with an explicit "No equipment" choice) or send an empty list, and make plan generation handle an empty list by excluding equipment-dependent exercises like Max Hangs. Tell me which you chose.
3. DOB: validate min year 1926 (and age 15-100) in JS on the client and again in the server schema (zod or equivalent) with a clear error message.
4. Injuries: maxlength 500 in the UI with a character counter, and the same limit in the server schema (reject or truncate with a 400 message). Strip control characters.
5. Days: show a short helper message ("Pick at least one training day") when the user tries to deselect the last day.
6. Add server-side validation for all onboarding fields so the API cannot be called with values the UI forbids.

Must NOT do: do not change the plan generation logic beyond handling empty equipment; do not change the onboarding step order or the number of steps; do not delete existing profiles or migrate saved data; do not loosen the age minimum (15 matches the Terms and Privacy Policy).

Done when: Continue is disabled until grade is chosen; no hidden equipment default; 1850 DOB is rejected client- and server-side; oversized injuries text is rejected; direct API calls with invalid values return 400.
```

## Prompt 14: Copy and text bugs

```text
Context: (1) "Set a start time for each of your 1 training day" (pluralization). (2) The gate toast says "Check your inbox for the verification link" but the flow uses a 6-digit code. (3) The verification email says "It expires in 10 hours"; this is probably meant to be minutes. (4) /verify-email-sent and /verify-email show the placeholder "your email address" when logged out instead of redirecting. (5) A weak or invalid signup body returns the bare API message "Invalid request body", shown unchanged with no explanation of the password or email rule.

Task: Fix these five text and flow issues.

Recommended fix:
1. Pluralization: "1 training day" / "N training days" using a tiny helper; check other count strings too.
2. Change the toast and any other "link" wording to "code" where the flow uses a code. Grep for "verification link" and "link" in auth flows.
3. Look at the actual code expiry in the server (do not guess): make the email text match the real value. If the real value is 10 hours and that is intended, keep it and tell me; if it is 10 minutes, fix the email text.
4. On /verify-email-sent and /verify-email, if there is no email in state or session, redirect to /enter (or /login).
5. Return field-level validation errors from the signup endpoint ({"error":"...","fields":{"password":"At least 8 characters, ..."}}) using the real rules, and show them next to the fields. Show the password rule under the field before the user types.

Must NOT do: do not change password rules, code length, or code lifetime; do not change the verification API contract for the existing mobile apps except by ADDING a "fields" property; do not rewrite unrelated emails or copy.

Done when: all five items behave as described and I can verify each on the live site.
```

## Prompt 15: Performance and hygiene

```text
Context: (a) GET /api/auth/user is called about 4 times per onboarding step, and GET /api/subscription about 5 times when loading /subscribe. (b) On every non-home route Chrome warns hero-bg.webp was preloaded but not used. (c) Malformed JSON bodies return a raw Express HTML "Bad Request" page instead of JSON. (d) The HTML document has no CSP, X-Frame-Options or Referrer-Policy headers (X-Frame-Options: DENY exists only on API responses); X-Powered-By: Express is exposed; two different HSTS headers are sent on API responses. (e) Full-plan nutrition targets were identical (2500 kcal / 160 g / 300 g / 2.5 L) for a user with no body data.

Task: Fix (a)-(d). For (e) only investigate and report.

Recommended fix:
1. (a) Deduplicate: use one shared query for user and subscription with a sensible staleTime (for example 30-60 s) and refetchOnWindowFocus only; remove components that fetch the same endpoint independently.
2. (b) Remove the hero-bg.webp preload from the global index.html and preload it only on the home route (or add the correct "as" and use it right away).
3. (c) Add an Express error handler that catches body-parser errors and returns 400 JSON {"error":"Invalid JSON"}.
4. (d) Add helmet (or manual headers) for HTML responses: X-Frame-Options SAMEORIGIN, Referrer-Policy strict-origin-when-cross-origin, X-Content-Type-Options nosniff, and app.disable("x-powered-by"). Send exactly one HSTS header. Introduce CSP in REPORT-ONLY mode first (Content-Security-Policy-Report-Only) allowing Stripe, Google and Apple sign-in domains, and tell me the violations you see; do NOT enforce it yet.
5. (e) Do not change anything. Tell me whether the nutrition targets are computed from user data or hardcoded, and where.

Must NOT do: do not enforce a strict CSP (it will break Stripe/Google/Apple); do not set X-Frame-Options DENY on the HTML if the app is embedded anywhere; do not disable auth or Stripe scripts; do not change caching of hashed assets.

Done when: fewer duplicate calls in the network tab, no preload warning on non-home routes, malformed JSON returns JSON 400, headers are present and single, and the app (including Stripe redirect and social login) still works.
```

## Prompt 16: Routing

```text
Context: Home "SUBSCRIBE" (Pro/Full) and "TRY IT FREE" both go to /onboarding, so the chosen plan and billing cycle are lost. An authenticated user can still open /login and is not redirected to the dashboard. /pricing is a 404 (the home button scrolls to an anchor, so this only affects external links).

Task: Fix these three routing issues.

Recommended fix:
1. Pass the selection through: /onboarding?plan=basic|pro|free&billing=monthly|yearly (use the internal ids), store it in sessionStorage, and pre-select that plan on /subscribe after onboarding (still let the user change it and still require an explicit click to activate).
2. On /login, /signup and /enter, if the user is authenticated, redirect (replace) to /dashboard, or to /subscribe if the plan is none, or to /onboarding if there is no profile.
3. Add a /pricing route that redirects to "/#pricing" (or scrolls to the anchor).

Must NOT do: do not auto-activate a plan from the URL; do not trust the query parameter (validate against the known plan ids); do not break the returnTo handling (it correctly rejects external URLs; keep that).

Done when: the chosen plan is pre-selected after onboarding, logged-in users cannot see the login form, and /pricing works.
```

## Prompt 17: /subscribe ignores the server-saved profile

```text
Context: After onboarding is completed and the profile is saved (GET /api/profile returns it), opening /subscribe in a new browser session or after clearing site data, with no plan yet, sends the user to /onboarding step 1 ("One more step: complete your profile"). The code checks browser-local onboarding data and plan state, not the server profile. Users who onboard on one device and continue on another must redo all 8 steps.

Task: Use the server profile as the source of truth.

Recommended fix:
1. In the /subscribe guard, call GET /api/profile (react-query). While loading, show a skeleton. If it returns a profile, allow the page. Redirect to /onboarding only on 404 (no profile).
2. Keep localStorage onboarding data only as a draft cache to resume an unfinished form, never as the completeness check.
3. When onboarding completes, make sure the profile is saved on the server BEFORE navigating to /subscribe, and invalidate the profile query.
4. Apply the same rule to every guard that decides "has this user onboarded" (dashboard, plan-ready, generating).

Must NOT do: do not change the onboarding form itself; do not create empty profiles; do not remove the local draft cache; do not skip the redirect for users who truly have no profile.

Done when: complete onboarding on browser A, open /subscribe in a fresh browser B logged in as the same user, and the plan choices show without repeating onboarding; a user with no profile is still sent to onboarding.
```

## Prompt 18: Free user without a profile hits a dead end

```text
Context: An account with plan free and no profile (reachable via activate-free, or after cancelling and clearing data) that opens /onboarding is redirected to /dashboard. The dashboard then shows either the full-screen "Building Your Plan…" spinner (no sidebar) or an empty page with only the sidebar, while the request storm from bug 2 runs in the background. The user cannot complete onboarding and sees no error.

Task: Let users with no profile (any plan) reach and complete onboarding, and make the dashboard handle the state.

Recommended fix:
1. Remove the rule that redirects users with a plan to /dashboard from /onboarding. Redirect away from /onboarding only when a profile already exists on the server (GET /api/profile succeeds).
2. Use one shared guard (for example useOnboardingState) returning "loading | needs-onboarding | ready | generating | failed" and use it in /onboarding, /dashboard, /subscribe so the routes agree.
3. When the state is needs-onboarding, /dashboard redirects to /onboarding (see also bug 2).
4. After onboarding completes for a user who already has a plan (including Free), generate the plan (POST /api/training/plan/generate) once and show the dashboard, without asking them to choose a plan again.
5. Never show a spinner with no exit: always render an error state with "Try again" and "Redo onboarding".

Must NOT do: do not let users without a profile access training features; do not create empty profiles; do not remove the redirect for users who already have a profile; do not change activate-free.

Done when: a Free account with no profile can open /onboarding, complete it, and reach a working dashboard; no state leaves the user on an endless spinner or empty page.
```

## Prompt 19: Nav bar shown on the choose-a-subscription page for brand-new accounts

```text
Context: A user who just created an account (verified, no plan yet) lands on /subscribe and sees the full app navigation: sidebar on desktop (Dashboard, AI Coach, Weekly Check-in, Nutrition, Subscription, Settings, "Get the app" box, sign-out) and the bottom tab bar on mobile. They have no plan or dashboard yet, so every nav item is a dead end.

Task: Show /subscribe without the app navigation while the user has no plan.

Recommended fix:
1. Find the layout wrapper that adds the sidebar and bottom tabs. Make it conditional: when the subscription plan is "none" (no plan, not even Free), render /subscribe in a minimal layout: logo, page content, and a small "Sign out" link only.
2. While subscription data is loading, render the minimal layout (never flash the nav then remove it).
3. Once a plan (including Free) is active, restore the normal layout. Users who already have a plan and visit /subscribe to change plan keep the normal nav.
4. Apply the same minimal layout to other pre-plan screens (onboarding, generating, plan-ready, verify pages) if they currently show the nav; list what you changed.
5. Make sure the desktop sidebar spacing (left padding) and mobile bottom padding are removed in the minimal layout so nothing is offset.

Must NOT do: do not remove the nav for users who have a plan; do not hide the nav based on the URL alone (it must depend on plan state); do not remove the sign-out option; do not change the plan cards.

Done when: a brand-new account sees no sidebar or bottom tab bar on /subscribe (desktop and mobile); after choosing any plan, including Free, the normal nav appears; existing subscribers are unchanged.
```
