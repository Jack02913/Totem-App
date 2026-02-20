# Totem — Onboarding Feature List
## Flow Logic, Algorithms & Data Dependencies

> **Scope:** The pre-app onboarding experience. 8 screens across 3 phases. Covers new user registration, bank connection via Basiq, pay cycle setup, data processing, and the first-task redirect. All amounts AUD/NZD.

---

## Overview: Onboarding Arc

```
Phase 1 — Hook (pre-auth)
  Screen 1: Welcome / Value prop
  Screen 2: Account creation

Phase 2 — Connect (the commitment)
  Screen 3: Bank selection
  Screen 4: Pay cycle setup
  Screen 5: Bank auth (Basiq OAuth)
  Screen 6: Data processing animation

Phase 3 — Reveal + First Task
  Screen 7: The Reveal (dashboard preview)
  Screen 8: Transaction review onboarding

Phase 4 — Progressive (in-app, no dedicated screen)
  Goal nudge: triggered on first Goals tab visit
  Insights nudge: triggered on first Insights tab visit
```

---

## 1. Welcome Screen

### What It Is
The first screen a new user sees. Must communicate the core value proposition within 3 seconds. Two CTAs: start the full flow, or explore with sample data immediately.

### "Explore with Sample Data" Path
```
User taps "Explore with sample data"
→ No account created
→ App opens on Dashboard with Alex's sample data (pre-existing mock dataset)
→ A subtle "Connect your bank to see your numbers" card appears in the dashboard
→ User can explore all 5 screens freely
→ When they tap "Connect your bank" anywhere: jumps to Screen 3 (bank selection)

Note: explore mode users are NOT stored in the database. A user record is only
created when they complete Screen 2 (account creation) or tap "Connect" from explore mode.
```

### "Get Started" Path
```
→ Proceeds to Screen 2 (account creation)
```

### Data Dependencies
- None (pre-auth)

---

## 2. Account Creation

### What It Is
Registration form. Supports Sign in with Apple (primary CTA) and email/password (secondary). Minimal fields — first name, email, password only.

### Sign in with Apple
```
Apple auth flow:
  → user_id from Apple → create users record with:
      first_name from Apple identity token
      email from Apple (may be relay email)
      auth_provider = 'apple'
  → Skip email verification
  → Go to Screen 3
```

### Email/Password
```
Validation:
  first_name: required, min 1 char
  email: valid email format, unique (check against existing users)
  password: min 8 chars, at least 1 number

On submit:
  → Create users record (onboarding_step = 'bank_connect')
  → Send verification email (async, non-blocking — user continues)
  → Go to Screen 3
```

### Edge Cases
| Scenario | Handling |
|---|---|
| Email already exists | "An account with this email already exists. Sign in instead." |
| Apple auth cancelled | Return to Screen 2, no error shown |
| Weak password | Inline validation, button disabled until valid |
| No internet | Toast: "Check your connection and try again" |

### Data Dependencies
- `users` (create record on submit)
- Apple identity token (for Sign in with Apple)

---

## 3. Bank Selection

### What It Is
Presents the major AU banks as selectable cards. Tapping a bank selects it (visual highlight) and enables the "Continue" button. Uses Basiq as the underlying connector for all banks.

### Banks Displayed
| Bank | Display | Basiq Connector |
|---|---|---|
| ANZ | 🔴 ANZ | `anz-personal` |
| CommBank | 🟡 CommBank | `cba-personal` |
| Westpac | 🔴 Westpac | `westpac-personal` |
| NAB | 🔴 NAB | `nab-personal` |
| Macquarie | 🟢 Macquarie | `macquarie-personal` |
| Other | 🏦 Other | Basiq bank search UI |

### "I'll connect later" Path
```
→ Users.onboarding_step = 'skipped_bank'
→ App opens with demo/sample data
→ "Connect your bank" persistent nudge in Dashboard
→ User can connect at any time from the Accounts screen
```

### Data Dependencies
- Basiq API (bank list, connector IDs)
- `users` (update onboarding_step)

---

## 4. Pay Cycle Setup

### What It Is
Asks the user to self-report their pay cycle. Four options: Weekly, Fortnightly, Monthly, Irregular. No income amount is asked — this is detected automatically from the bank feed.

### Options
| Selection | Sets `users.pay_cycle_type` | Notes |
|---|---|---|
| Weekly | `weekly` | Asks for day of week |
| Fortnightly | `fortnightly` | Asks for which week (this week / alternate weeks) — default selected |
| Monthly | `monthly` | Asks for day of month |
| Irregular | `irregular` | No sub-question; activates irregular income mode |

### Day Sub-question (shown after selection, except Irregular)
```
"Which day do you typically get paid?"
  → Fortnightly/weekly: day picker (Mon–Sun)
  → Monthly: date picker (1–28, with "last day of month" option)
```

This is stored as `users.pay_cycle_day_of_week` or `users.pay_cycle_day_of_month`.

### Auto-Refinement (background)
After bank data is pulled in Screen 6:
```
Pay cycle detection algorithm runs (see Dashboard/feature-list.md Section 3)
  IF detected cycle matches user-reported cycle → confirm, no change
  IF detected cycle differs → update pay_cycle_type to detected value
                            → flag for user confirmation on first dashboard view
                            → show "We detected your pay cycle is [X]. Is that right?" card
```

The user is never asked their income amount — it's derived from the income transaction amounts.

### Data Dependencies
- `users` (pay_cycle_type, pay_cycle_day_of_week, pay_cycle_day_of_month)

---

## 5. Bank Auth (Basiq OAuth)

### What It Is
Launches the Basiq-hosted authentication flow in an in-app browser. The user logs into their bank directly — Totem never sees credentials. On success, Basiq returns a connection ID.

### Flow
```
1. Totem backend creates a Basiq user_id (if not already exists) and requests a consent token
2. In-app WebView opens: Basiq hosted auth UI for the selected bank
3. User authenticates with their bank credentials directly with the bank
4. Bank issues CDR consent → Basiq receives auth code
5. Basiq returns connection_id and account list to Totem callback
6. Totem stores connection_id, fetches initial account list
7. WebView closes → user returned to Screen 6 (Processing) automatically
```

### Security Notes
- Totem never stores or sees bank credentials
- CDR consent is revocable at any time from Basiq or from within the bank's own app
- Read-only consent — Totem cannot initiate transfers or payments
- Consent tokens stored encrypted server-side only

### Error States
| Error | User-facing message |
|---|---|
| Bank authentication failed (wrong password) | "Authentication failed. Try again or choose a different bank." |
| Bank system unavailable | "ANZ is temporarily unavailable. Try again in a few minutes or connect a different bank." |
| Basiq API error | "Connection failed. Our team has been notified. Try again shortly." |
| User cancels in WebView | Return to Screen 5 with "Connection cancelled" message |

### Data Dependencies
- Basiq API (consent token, OAuth flow, connection_id)
- `accounts` (populated from Basiq account list on success)
- `account_connection_log` (first entry inserted on success)

---

## 6. Data Processing

### What It Is
A full-screen animated processing view shown while Basiq pulls transaction history and the AI categorisation pipeline runs. Auto-advances to Screen 7 when complete.

### Processing Steps (displayed to user)
```
1. ✓ [Bank] connected — X accounts linked
2. ◉ Importing transactions — N of ~M imported (live counter)
3. ○ Categorising with AI — waiting… → active → all categorised
4. ○ Building your insights — waiting… → active → insights ready
```

### Backend Pipeline (triggered after Basiq connection)
```
Step 1: Account sync
  Basiq pulls all accounts → inserted into accounts table
  account_connection_log entry created

Step 2: Transaction import (async)
  Basiq pulls up to 12 months of transactions
  Inserted into transactions table (source = 'bank_feed')
  ai_categorisation_queue populated for each transaction
  Progress updates polled via SSE or polling endpoint (UI updates counter)

Step 3: AI categorisation (async, runs in parallel with import)
  Categorisation worker processes ai_categorisation_queue
  Rules applied first, then AI model
  transaction_reviews rows created for each non-rule-matched transaction

Step 4: Metrics computation
  pay cycle detection runs → may update users.pay_cycle_type
  recurring bill detection runs → recurring_bills populated
  lifestyle_creep_snapshots populated for all historical months
  computed_metrics_cache populated for all dashboard metrics
  forecast_cache populated for next 3 months
  metric_snapshots populated for last 12 months

Step 5: Webhook to frontend
  Push notification (or SSE event): "processing_complete"
  Frontend receives event → auto-advances to Screen 7
```

### Timing
```
Typical: 8–20 seconds
Slow (large history, slow Basiq): up to 45 seconds
Timeout: If > 60 seconds → show "Taking longer than expected, we'll notify you when ready"
         → Allow user to proceed anyway → processing continues in background
```

### Data Dependencies
- Basiq API (transaction pull)
- `transactions` (write target)
- `ai_categorisation_queue` (write target)
- `transaction_reviews` (created by categorisation worker)
- `recurring_bills` (created by bill detection)
- `lifestyle_creep_snapshots` (created for all historical months)
- `computed_metrics_cache`, `forecast_cache`, `metric_snapshots` (created)

---

## 7. The Reveal

### What It Is
The highest-converting moment in onboarding. Shows the user a preview of their actual dashboard data — Safe to Spend, top category, savings rate, and one AI flag — immediately after processing completes.

### Data Displayed
```
All values pulled from computed_metrics_cache + transactions:

Safe to Spend: computed from pay period data (see Dashboard formula)
Top Category:  category with highest spend in current calendar month
Savings Rate:  (income - spend) / income × 100 for last completed pay period
AI Flag:       highest-priority insight from ai_insights (generated during processing)
               If Lifestyle Creep detected: always show this (high emotional salience)
```

### "Does this look right?" Decision
```
"Yes, let's go →"
  → Update users.onboarding_step = 'review_transactions'
  → Go to Screen 8

"Something looks off — I'll fix it"
  → Also goes to Screen 8 (the fix happens through the review process)
  → A note card in Screen 8: "We'll learn from your corrections"
```

### Edge Cases
| Scenario | Handling |
|---|---|
| Processing took > 45s and user skipped | Show a reduced preview: just top category + estimated Safe to Spend |
| First pay not yet detected | Safe to Spend shows estimated figure with "(estimated)" label |
| All transactions auto-categorised by rules | Review count already > 0; show "N already categorised" in Screen 8 |

### Data Dependencies
- `computed_metrics_cache` (Safe to Spend, Savings Rate)
- `transactions` + `categories` (top category)
- `ai_insights` (AI flag)
- `lifestyle_creep_snapshots` (for lifestyle creep flag)

---

## 8. Transaction Review Onboarding

### What It Is
Shows the 30-transaction progress bar (starting at 0/30 for new users) and a preview of 3 transactions from the To Review queue. Bridges the user from onboarding into the ongoing app experience.

### Queue Preview (3 items shown)
```
SELECT tr.*, t.merchant_name, t.amount, t.date, c.name, c.icon, tr.ai_confidence_label
FROM transaction_reviews tr
JOIN transactions t ON t.id = tr.transaction_id
JOIN categories c ON c.id = tr.ai_suggested_category_id
WHERE tr.user_id = $1 AND tr.review_status = 'pending'
ORDER BY tr.ai_confidence_label DESC, t.amount DESC
LIMIT 3
```

Shows the 3 highest-confidence, largest-amount pending reviews to make approval as frictionless as possible.

### CTAs
```
"Start reviewing →"
  → Exits onboarding
  → Navigates to Accounts screen, To Review tab
  → Onboarding considered complete: users.onboarding_complete = true

"Skip for now, take me to the app"
  → Exits onboarding to Dashboard (home screen)
  → users.onboarding_complete = true
  → A nudge card appears in Dashboard: "Review transactions to unlock AI features"
```

### Data Dependencies
- `transaction_reviews` (queue preview)
- `transactions`, `categories` (review item data)
- `users` (mark onboarding_complete = true on exit)

---

## 9. Progressive In-App Onboarding (Phase 4)

These are not screens — they are contextual nudges delivered via the AI chat bubble (✨ FAB) the first time a user visits each screen.

### Goals Tab — First Visit Nudge
```
Trigger: user navigates to Plan screen → Goals tab, AND user_ai_activation.milestone_type
         'thirty_transactions' NOT YET reached

Chat bubble animates open with message:
  "Want to set up a savings goal? I can suggest some based on your spending pattern
   once you've reviewed a few more transactions. [X left to go!]"

CTA in chat: "Set up a goal now" → opens Add Goal sheet
             "Remind me later"   → dismisses, does not show again for 7 days

Trigger: user navigates to Plan screen → Goals tab, AND milestone IS reached

Chat bubble:
  "Your AI features are unlocked 🎉 Want me to suggest some goals based on your
   spending? I can see some patterns worth talking about."

CTA: "Yes, show me" → opens AI goal suggestions
     "Not now"      → dismisses
```

### Insights Tab — First Visit Nudge
```
Trigger: first time user navigates to Insights, AND at least 14 days of real data exists

Chat bubble:
  "Here's what I found in your first [X] days. Your top insight: [top ai_insight body_text]"
```

### Nudge State Tracking
```
Stored in user_settings:
  'onboarding_goals_nudge_shown':    true/false
  'onboarding_goals_nudge_dismissed_at': timestamptz
  'onboarding_insights_nudge_shown': true/false
```

---

## Appendix: Onboarding State Machine

| State (`users.onboarding_step`) | Screen | Next |
|---|---|---|
| `null` / new | Screen 1: Welcome | `account_created` |
| `account_created` | Screen 2: Sign Up | `bank_connect` |
| `bank_connect` | Screen 3: Bank Selection | `pay_cycle` |
| `pay_cycle` | Screen 4: Pay Cycle | `bank_auth` |
| `bank_auth` | Screen 5: Bank Auth | `processing` |
| `processing` | Screen 6: Processing | `reveal` |
| `reveal` | Screen 7: The Reveal | `review_transactions` |
| `review_transactions` | Screen 8: First Review | `complete` |
| `complete` | Main app | — |
| `skipped_bank` | Main app (sample data) | Nudge to connect |

## Appendix: Calculation Run Schedule

| Feature | Trigger |
|---|---|
| Account creation | On form submit |
| Basiq user + consent token | On bank selection + "Continue" |
| Transaction import | Immediately after Basiq OAuth success |
| AI categorisation | Concurrent with import (pipeline) |
| Pay cycle detection | After import complete |
| Recurring bill detection | After import complete |
| Historical snapshots | After import complete (batch) |
| Dashboard metrics | After all import + categorisation complete |
| Progressive nudges | On first visit to each relevant screen |
