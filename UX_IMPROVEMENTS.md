# ClimbSmarter UI/UX review (Apple design principles)

**Reviewed:** 2026-10-08. Live site (home, `/enter`, `/signup`, mobile 390px and desktop 1280px) plus the client source from the 2026-10-02 export. The signed-in screens (dashboard, coach, settings) were reviewed from code only, not from screenshots, because I have no test login in this session.
**Lens:** the "Apple design" guidance: instant press feedback, interruptible spring motion, spatial consistency, translucent materials, size-aware typography, reduced-motion support, and the eight principles (purpose, agency, familiarity, simplicity, craft and so on).
**Tags:** **(certain)** = seen in the live site or code; **(likely)** = strong inference; **(guessing)** = judgement call.

## What is already good

- Dark, high-contrast theme with one strong accent colour; the orange on near-black is legible and distinctive. (certain)
- The auth screens are clean and short: social sign-in first, email second. (certain)
- Honest copy ("There are no made-up reviews here") builds trust better than invented testimonials. (certain)
- Buttons are 48 to 56px tall, so touch targets are fine. (certain)
- Cards and panels already use a translucent "glass" style (`glass-panel`), so the material language exists. (certain)

## Findings, most important first

| # | Finding | Evidence | Principle |
|---|---|---|---|
| 1 | **Home headline reads "Your plan.Your results." on mobile**: the line break is hidden on small screens and no space replaces it. A visible typo in the biggest text on the page. | Live screenshot at 390px; `pages/Home.tsx` line 244 uses `<br className="hidden sm:block" />` with no space around it. (certain) | Craft |
| 2 | **No reduced-motion support anywhere.** There are zero `prefers-reduced-motion` rules or `useReducedMotion` calls, yet the app uses framer-motion, `tw-animate-css`, and a 38-second infinite marquee. | grep over `src` = 0 hits. (certain) | Accessibility |
| 3 | **Bottom tab labels are 9px.** Below any readable size, in all caps with wide tracking. | `components/layout/AppLayout.tsx` line 275: `text-[9px] ... uppercase tracking-wider`. (certain) | Craft, legibility |
| 4 | **ALL-CAPS wide-tracked text is the default for every button** (116 `uppercase` uses). It is loud, slow to read, and is the cause of the clipped plan-card buttons in the bug report (#9). Apple uses sentence case and reserves caps for tiny labels. | `Button.tsx`: `uppercase tracking-wider` on the base style. (certain) | Simplicity |
| 5 | **Press feedback is slow.** Buttons use `transition-all duration-300`, so the 0.98 "press" scale takes 300ms to appear. Feedback should be near-instant on pointer-down. 62 elements use `transition-all`, which also animates layout properties unnecessarily. | `Button.tsx`; live computed style showed `0.3s`. (certain) | Response |
| 6 | **Vague or duplicated labels.** Header button says "LOG IN / SIGN UP" while the hero has a second "MEMBER LOGIN" button right below "BUILD MY PLAN". Under the social buttons the email button just says "CONTINUE". | Live screenshots. (certain) | Specific labels, simplicity |
| 7 | **Hero wastes the first screen.** About 270px of empty space sits below the hero buttons on a phone, and the hero photo is so dark it is barely visible, so nothing shows what the product actually looks like. | Live screenshot `h0`. (certain for the gap; likely for the cause) | Purpose, show the common path |
| 8 | **Three font families loaded** (Inter, Manrope, Outfit) while the CSS uses only Manrope and Outfit. Extra download for no visible benefit. | `index.html` lines 33 to 35 vs `index.css`. (likely: I did not check every file for Inter) | Craft, performance |
| 9 | **Typography uses one tracking rule for everything.** Headings get `tracking-tight`, buttons get `tracking-wider`, nothing is tuned by size. Large display text wants slightly negative tracking, small text slightly positive. | `index.css`, `Button.tsx`. (certain) | Typography |
| 10 | **Focus ring offset is the default white** on a dark background (`ring-offset-2`), which makes a white halo around focused buttons instead of a clean orange ring. | `Button.tsx`. (likely: depends on the ring-offset colour token) | Craft |
| 11 | **Modals and sheets** use fade-style transitions from the component library, not interruptible springs, and are not anchored to their trigger. No swipe-down-to-dismiss on mobile sheets. | `components/ui/dialog.tsx`, `drawer.tsx`, `sheet.tsx` exist; no gesture code found. (likely) | Interruptibility, spatial consistency |
| 12 | **Chrome is opaque in places** where Apple uses translucent materials with content scrolling underneath (header, bottom tab bar). The home header is already close to this. | Live screenshot; `AppLayout.tsx`. (likely) | Materials |

## What I recommend (in this order)

1. **Quick wins (small, safe, high visible impact):** fix the headline space (#1), raise tab labels to at least 11px (#3), switch buttons to sentence case with normal tracking and fix the labels in #6, make press feedback 100ms and animate only the properties that change (#4, #5), fix the focus ring (#10), drop the unused font (#8), tighten the hero spacing (#7).
2. **Accessibility pass:** reduced motion, reduced transparency, and higher contrast media queries (#2).
3. **Typography tokens:** size-aware tracking and line height (#9).
4. **Materials and navigation:** translucent header and tab bar with scroll-edge fades instead of hard dividers (#12).
5. **Motion:** spring-based, interruptible modals and sheets with swipe to dismiss, and directional onboarding step transitions (#11).

Do these one at a time. A visual change is a taste call, and each one is easy to review in a screenshot. Nothing here changes pricing, plans or any server behaviour.

---

# Replit prompts (five, in the order above)

Paste the **Shared rules** block first, then one prompt per run. Do not send them together: prompts 1 and 4 touch the same layout files.

## Shared rules

```text
You are making a UI polish change to ClimbSmarter.app (React, Tailwind, framer-motion, wouter). Read the whole request before touching code.
- Change ONLY what the request lists. No refactors, no dependency upgrades, no new dependencies unless named, no server changes.
- Apply changes to the web client that serves climbsmarter.app in production (artifacts/clmbsmarter) and tell me whether the other client (clmbsmarter-v2) shares the same code and was affected.
- Do NOT change prices, plan names, plan logic, test mode, Stripe, or the database. Do NOT change copy other than the labels the request names.
- Keep the dark theme, the orange primary colour and the existing brand fonts (Manrope and Outfit) unless the request says otherwise.
- Do NOT publish or deploy.
- Verify in the running app at 320, 390, 768 and 1280px. Report exact steps and results with before and after screenshots. Run the type check, tests and build. Say plainly what you could not verify.
- End with: files changed, one sentence per change, and anything you noticed but did not change.
```

## UX-1: Quick wins

```text
Make these small fixes:
1. Home page headline: on mobile it reads "Your plan.Your results." because the line break is hidden below the sm breakpoint and no space replaces it. Make it read "Your coach. Your plan. Your results." with proper spaces on every width, with the existing line break behaviour kept on larger screens.
2. Bottom tab bar labels are 9px, uppercase, wide tracking. Make them at least 11px, sentence case, normal tracking, and keep them on one line (shorten "Weekly Check-in" to "Check-in" in the tab bar only if it does not fit; keep the full name in the desktop sidebar).
3. Buttons: remove the forced uppercase and wide letter spacing from the base Button style so labels are sentence case ("Build my plan", "Log in"). Keep uppercase only for small eyebrow labels above headings. After this, check the plan cards on the subscribe page: the long call-to-action labels must no longer clip at any width.
4. Press feedback: change the button transition from "transition-all duration-300" to transitions of only background, border, box-shadow and transform; make the pressed (active) state respond within about 100ms and keep hover transitions around 200ms. Search for other uses of "transition-all" on interactive elements and apply the same rule only where it is clearly a button or link.
5. Labels: the home header button says "LOG IN / SIGN UP"; change it to "Log in". The hero has a second "MEMBER LOGIN" button directly under "Build my plan"; replace it with a small text link "Already a member? Log in". Under the Google and Apple buttons on the enter and sign-up pages, change the email button label from "Continue" to "Continue with email".
6. Focus ring: make the keyboard focus ring a clean 2px orange ring with an offset that uses the page background colour (not white).
7. Fonts: Inter is loaded in the HTML but the app uses Manrope and Outfit. Check whether Inter is used anywhere; if it is not, stop loading it.
8. Hero: on a 390 by 844 phone there is about 270px of empty space below the hero buttons. Tighten the hero so the first screen is filled (for example vertically balanced content or a visible product teaser), and make the hero background image visible enough to read as an image, without lowering text contrast below WCAG AA.
Must NOT do: do not change any other copy, colours, spacing outside the hero, or button sizes (keep 48 to 56px height).
```

## UX-2: Accessibility

```text
Add reduced-motion, reduced-transparency and high-contrast support:
1. prefers-reduced-motion: reduce - stop the infinite marquee on the home page (show it static or let the user scroll it), remove slide, spring, parallax and overshoot effects, and replace them with short opacity cross-fades (about 150 to 200ms). Apply this globally through a CSS rule and through a small shared helper for framer-motion (use its reduced motion hook) so every animated component respects it. Keep opacity and colour changes that help comprehension.
2. prefers-reduced-transparency: reduce - make the translucent "glass-panel" surfaces and any blurred header or tab bar use a solid background and no backdrop blur.
3. prefers-contrast: more - make muted text and borders higher contrast (near-solid backgrounds, visible borders).
4. Avoid full-viewport moving backgrounds and abrupt brightness jumps.
Must NOT do: do not remove animations for users who have not asked for reduced motion; do not change colours for normal users.
Verify by emulating each media feature in the browser and show screenshots of the home page, subscribe page and dashboard in each mode.
```

## UX-3: Typography tokens

```text
Introduce size-aware typography using CSS custom properties or Tailwind theme tokens, without changing fonts:
- Display and large headings: slightly negative letter spacing (about -0.02em) and tight line height (about 1.05 to 1.15).
- Medium headings: about -0.01em and line height about 1.2.
- Body text: letter spacing 0 and line height about 1.5.
- Small and caption text (below 13px): slightly positive letter spacing (about 0.01em), never below 11px.
- Eyebrow labels (the small orange section labels): keep uppercase with modest tracking (about 0.08em).
Apply the tokens to the existing heading and body styles so pages pick them up without editing every page. Use rem units so text scales with the user's browser text-size setting and check that nothing breaks at 200% text size.
Must NOT do: do not change font families or weights; do not change copy; do not touch the logo.
```

## UX-4: Materials and navigation

```text
Make the app chrome feel like layered, translucent material:
1. Header (home and app) and mobile bottom tab bar: translucent background with a backdrop blur and saturation boost, a faint bright top edge on the tab bar, and content scrolling underneath. Remove hard 1px divider lines under sticky headers; instead fade a soft gradient or blur mask only where scrolling content meets the bar.
2. Material weight: heavier (darker, stronger blur and shadow) for large structural surfaces such as the sidebar, lighter for small interactive elements. Never stack one light translucent surface on another.
3. Text over translucent surfaces must stay readable: use higher-contrast text colour and slightly heavier weight rather than muted grey, and keep colour on solid layers.
4. Respect the reduced-transparency setting from the accessibility change (solid fallback).
5. Make sure the bottom tab bar respects the phone's bottom safe area and never covers page content (add matching bottom padding).
Must NOT do: do not change the navigation items or their order; do not change which items are shown to which plan.
```

## UX-5: Motion for modals, sheets and onboarding

```text
Make interactive motion physical and interruptible using framer-motion springs:
1. Default spring for UI: critically damped (no overshoot), response about 0.35s. Use a little bounce (about 0.2) only when the motion follows a user flick or drag.
2. Modals and bottom sheets (feedback, share, the preview unlock dialog, and any sheet or drawer): animate from the trigger (set the transform origin to the button that opened it), exit along the same path they entered, and never block input during the animation (the user can grab or close it mid-motion). On mobile sheets, add swipe-down-to-dismiss that follows the finger 1:1 and, on release, uses the finger's velocity to decide whether to close, with a soft rubber-band resistance when dragged upward past the top.
3. Onboarding steps: the next step slides in from the right and the previous from the left (and the reverse on Back), with the same spring. Keep the step indicator in sync.
4. Respect the reduced-motion setting from the accessibility change (cross-fade instead).
5. Press feedback on cards and list rows that are tappable: instant scale-down on pointer down, spring back on release.
Must NOT do: do not add new libraries; do not change onboarding questions, validation or the order of steps; do not use CSS keyframes for gesture-driven motion.
```
