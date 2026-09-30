# ClimbSmarter.app: Bug Report (no-card subscription testing)

**Tested:** 2026-09-30, production (`https://climbsmarter.app`)
**Method:** Headless Chromium (Playwright), desktop 1280px and mobile 390px viewports, plus direct API calls made from the logged-in browser session. The client bundle was read to learn endpoints and validation rules.
**Confidence tags:** **(certain)** = reproduced with captured evidence; **(likely)** = strong inference; **(guessing)** = unverified.

## Headline

**The no-card change is incomplete.** Test mode is on (`testModeEnabled: true`), and Pro and Full activate with no card. But a live Stripe checkout can still be created, and most marketing copy still says users are charged today. Bugs 1 and 3 are the ones to fix before you show this to anyone.

## Summary

| # | Severity | Area | Bug |
|---|---|---|---|
| 0 | **High** | Post-onboarding | Dashboard stuck forever on a dark "Building Your Plan…" spinner screen after choosing a plan (the likely "black screen") |
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
| 16 | Low | Routing | Plan choice lost from home pricing; authed users can view `/login` |

---

## 0. Dashboard stuck on a near-black "Building Your Plan…" screen: High (certain)

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
