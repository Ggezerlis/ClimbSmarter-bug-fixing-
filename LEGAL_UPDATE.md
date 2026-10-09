# ClimbSmarter Terms and Privacy Policy: update review

**Prepared:** 2026-10-09. **Compared:** the Terms (effective 17/04/2026) and Privacy Policy (effective 17/04/2026, last updated 01/08/2026) against the app code exported on 2026-10-09 (server, web client, database schema).
**This is a drafting and fact-checking aid, not legal advice.** The documents also contain a line saying the company should get professional legal review (Terms section 6). I agree with that, and I would do it before real payments start.
**Tags:** **(certain)** = confirmed in the code; **(likely)** = strong inference; **(guessing)** = needs your confirmation.

## 1. What is wrong or missing today

| # | Where | Problem | Evidence |
|---|---|---|---|
| 1 | Terms 5 | Says paid plans are "charged immediately" and that Free is "the only no-cost, no-card option". Today Pro and Full activate with no card (early access). **(certain)** | `lib/access.ts` `TEST_SUBSCRIPTIONS_ENABLED = true`; `/subscription/activate-test` |
| 2 | Terms 5 and 14 | Nothing says what early access is, how it ends, or that test plans return to Free. **(certain)** | `access.ts`: test plans fall back to Free when the flag is off; cancelling a test plan returns to Free at once |
| 3 | Terms 6 | The published text contains an internal note ("The Company should obtain professional legal review"). It should not be visible to users. **(certain)** | Terms.tsx section 6 |
| 4 | Terms 6 and checkout | The withdrawal-right text talks about "express consent" at checkout, but the Stripe checkout has no consent or acknowledgement step. **(certain)** | `stripe.ts` creates the session with `payment_method_collection: 'always'` only |
| 5 | Terms 7 | Says each 100-day milestone "adds 5%" up to 50%. The code compounds it: 5% of the price, then 5% of what is left. 2 milestones = 9.75%, not 10%. **(certain)** | `lib/pricing.ts` `getDiscountPercent`: `(1 - 0.95^tiers) * 100`, capped at 50 |
| 6 | Privacy 1 | Does not list **date of birth**. Onboarding stores it. **(certain)** | `onboardingValidation.ts` `dateOfBirth`, `age` |
| 7 | Privacy 1, 4 | Does not mention **injury and limitation notes** (free text, up to 500 characters). That is health information, which GDPR treats as a special category needing explicit consent. It is also sent to the AI provider as part of the coach and plan prompts. **(certain)** | `injuryLimitations`; `coachContext.ts` line 110 |
| 8 | Privacy 1 | Does not list **climb logs and set logs** (performance logging tables). **(certain)** | tables `climb_logs`, `set_logs` |
| 9 | Privacy 1, 11 | Does not mention **support emails** (sender address, message text, attachment file names), which are stored, and that **Anthropic (Claude)** is used to triage them. **(certain)** | `support_tickets` table; `lib/aiTriage.ts` uses `@anthropic-ai/sdk` |
| 10 | Privacy 11 | Lists OpenAI as the AI provider but the code reaches it through Replit's AI integration gateway (`AI_INTEGRATIONS_OPENAI_BASE_URL`), so Replit also handles those prompts. **(likely)** | `trainingPlanGenerator.ts`, `coach.ts` |
| 11 | Privacy 6, 8 and Terms 14 | Says account data is deleted "within 30 days". In the app, "Delete Account" deletes immediately. Support emails are **not** deleted with the account, and Stripe keeps its own customer record. **(certain)** | `DELETE /api/auth/account` (`auth.ts`); `support_tickets` has no user link |
| 12 | Privacy 6 | Says payment records are kept for up to 7 years. That period should be confirmed with an accountant. **(guessing)** | none in code |
| 13 | Privacy 5 | Transfers section names only SCCs and the EU-US Data Privacy Framework generally. It does not say which providers rely on which. **(likely)** | none in code |
| 14 | Both | No "Last updated" date on the Terms. The Privacy date (01/08/2026) is older than the changes in this review. **(certain)** | pages |
| 15 | Both | The operator "Ebusiness & Content Services" has no registered address, registration number or VAT number. **(certain)** | Contact sections |

**Sections that look accurate and can stay:** age 15 minimum with parental permission under 18 (matches Greece's age of digital consent, **(likely)**), email-first sign-in, cookies (essential only), analytics (daily-rotating hash, IP not stored, confirmed in `analytics.ts`), push notifications, workout sharing, governing law, severability.

## 2. Proposed wording (replace only these parts)

### Terms of Service

**Header:** change to `Effective Date: 17/04/2026 · Last Updated: [publication date]`.

**New section "Early access" (insert after section 5, renumber after it):**
> During early access, paid plans (Pro and Full) can be activated without entering payment details and are not charged. Early access has no fixed end date. We will give you at least [X] days' notice by email and in the Service before it ends. When it ends, your account returns to the Free plan and you keep your profile, training plans and history. You will only be charged if you choose to subscribe and enter payment details. You can return to the Free plan at any time from Settings.

**Section 5, replace these two paragraphs:**
- Replace "Free plan: Free is the only no-cost, no-card option..." with: "**Free plan:** Free has no cost and needs no payment details. It includes an AI-generated training plan, weekly check-ins and plan progression. Paid features require a paid plan or early access (see Early access)."
- Replace "Immediate billing and renewal: Paid subscriptions are charged immediately..." with: "**Billing and renewal:** When you subscribe with payment details, you are charged when you complete checkout. Subscriptions renew automatically at the monthly or yearly interval you selected until you cancel."

**Section 6 (EU right of withdrawal), replace the whole section:**
> If you are a consumer in the EU, you generally have 14 days to withdraw from a distance contract. When you subscribe, we ask you to confirm that you want access to the digital service to start immediately and that you understand how this affects your right of withdrawal. [Lawyer to confirm exact wording.] Nothing in these Terms limits consumer rights that cannot be waived by law.

*(And delete the sentence about obtaining legal review.)* **This needs the checkout step in section 4 below, otherwise the text is not true.**

**Section 7 (Streak discounts), replace the first two bullets:**
> - A discount is earned for every 100 consecutive days of training.
> - Each milestone reduces your price by 5% of the price that remains after earlier milestones. For example, two milestones give 9.75% off. The total discount is capped at 50%.

**Section 10 (AI-Generated Content), first sentence:** replace "powered by OpenAI" with "powered by third-party AI providers, currently including OpenAI".

**Section 14 (Account deletion):** add after the bullet list: "Deleting your account in the app takes effect immediately. If you contact us by email instead, we complete the deletion within 30 days. Emails you sent to support are deleted within [X months] of the end of the conversation."

**New section "Health information" (insert after section 4):**
> You may choose to tell us about injuries or physical limitations so your plan can take them into account. This is optional. It is health information, we use it only to personalise your plan and coaching, and we send it to our AI provider for that purpose. You can remove it at any time in Settings. The Service does not give medical advice.

**Contact (both documents):** add `Registered address: [ ]`, `Company registration number (GEMI): [ ]`, `VAT number: [ ]`, `Data protection contact: support@climbsmarter.app`. **I cannot invent these. You must supply them.**

### Privacy Policy

**Header:** set `Last Updated: [publication date]`.

**Section 1, add these items:**
- Under "Onboarding and Training Profile": **Date of birth and age**; **Injuries and physical limitations (optional free text)**; climbing environment; competition status.
- Under "Training and Usage Data": **Climb logs and set logs** (climbs and sets you record).
- New heading **Support Emails:** "When you email support@climbsmarter.app we store the sender address, the message text and attachment file names so we can answer you."
- Under "Cookies and Session Data": add "Your browser also stores a few items in local or session storage: a draft of your onboarding answers before you sign up, your chosen plan before checkout and short-lived flags that track plan generation. They stay on your device."

**Section 4 (Legal basis), add:**
> **Explicit consent (health information):** if you enter injuries or physical limitations, we process that text on the basis of your explicit consent, which you give by entering it and can withdraw by deleting it in Settings or by emailing us. **(Needs the consent notice in section 4 below.)**

**Section 11 (Third parties), replace the OpenAI line and add two lines:**
- "**OpenAI (through Replit's AI integration):** generates training plans and powers the AI Coach. Your profile (including injury notes if you entered them), your plan, recent workouts and relevant nutrition logs are sent to generate each response."
- "**Anthropic (Claude):** helps us triage emails you send to support. The message text is sent to Anthropic to help classify and draft a reply. A person reviews it before anything is sent."
- *(The human-review statement must be true. Replit's notes say replies need manual approval. Please confirm.)* **(guessing)**

**Section 5 (Transfers):** after verifying with each provider, name the safeguard used (for example "OpenAI, Anthropic, Stripe and Resend rely on SCCs or the EU-US Data Privacy Framework"). **I could not verify each provider's current status. Please check each provider's data-processing terms.**

**Sections 6 and 8 (Retention and deletion), replace "within 30 days":**
> Deleting your account in the app removes your profile, plans, logs, nutrition data, coach messages and sessions immediately. If you ask us by email, we do it within 30 days. Stripe keeps its own payment and customer records as required by law. Emails you sent to support are kept for [X months] and then deleted. Payment records are kept for [accountant to confirm] years.

**Section 7 (Rights), add:** "You can also complain to the Hellenic Data Protection Authority (dpa.gr)."

## 3. Code changes the wording depends on

1. **Checkout consent for the withdrawal right.** Before Stripe checkout, show a required checkbox: "I want my access to start now and I understand how this affects my right to withdraw." Record that the user ticked it. **(Terms section 6 is not true without it.)**
2. **Consent notice at the injuries field** on onboarding: "Optional. This is health information. We use it to personalise your plan and send it to our AI provider for that. You can remove it anytime in Settings." Make the field optional and never required.
3. **Support ticket cleanup:** either delete support tickets when an account is deleted (match the sender email to the account), or state the retention period above. Today they stay forever. **(certain)**
4. A "Remove injury notes" control in Settings, if one does not exist yet.

## 4. Decisions only you can make

- **Who is the operator?** The documents name "Ebusiness & Content Services" in Greece. The legal operator, the data controller and the holder of the Stripe account should be the same entity. Please confirm this and give me the registered address, registration number and VAT number.
- **Early access notice period** `[X]` days, and **support email retention** `[X]` months. My suggestion: 30 days and 24 months.
- **Payment record retention** (accountant).
- **Whether to keep 15 as the minimum age.** It matches Greek law; changing it is a legal decision.

## 5. Order I recommend

1. Get answers to section 4 and have a lawyer or accountant look at sections 1 to 3 once.
2. Apply the wording and the code changes (via Replit now, or by me through Git later).
3. Publish the Terms and Privacy **before** switching Stripe to live, so customers see the right terms on their first real payment.
4. Email existing users if changes are material (the Terms already promise this in section 19).
