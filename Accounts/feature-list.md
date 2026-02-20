# Totem — Accounts Feature List
## Math, Algorithms & Data Dependencies

> **Scope:** Accounts screen only (Screen 4). 4-tab layout: Accounts | To Review | Rules | Subs. All currency AUD/NZD. All amounts in cents unless noted.

---

## 1. Account Linking (Basiq / NZ Connector)

### What It Is
The mechanism by which a user connects their real bank accounts to Totem. Uses Basiq for Australian banks (Open Banking CDR) and an equivalent connector for NZ (Akahu). Once linked, transactions flow in automatically via webhook.

### Connection Flow
```
1. User taps "Add Account" → Basiq hosted auth UI launches (in-app browser / WebView)
2. User selects institution → authenticates with bank credentials
3. Basiq returns connection_id and account list
4. Totem stores account metadata in accounts table (see database-schema.md)
5. Basiq performs initial data pull — last 90 days of transactions
6. Totem runs initial categorisation batch on all imported transactions
7. Totem runs initial recurring bill detection on imported history
8. Connection status set to 'active'; live feed indicator activates
```

### Sync Frequency
| Trigger | Action |
|---|---|
| Basiq webhook (new transaction) | Insert transaction, run categorisation + bill detection |
| App open | Trigger Basiq refresh if last sync > 4 hours ago |
| Manual pull-to-refresh | Force Basiq refresh, regardless of last sync time |
| Daily scheduled (02:00) | Balance reconciliation — ensure account.balance matches Basiq |

### Live Feed Indicator
- Pulsing green dot displayed on the Accounts tab header and next to each account row
- Dot is green + pulsing if `last_synced_at > NOW() - INTERVAL '4 hours'`
- Dot is amber (no pulse) if `last_synced_at BETWEEN NOW() - 24h AND NOW() - 4h`
- Dot is red (no pulse) if `last_synced_at < NOW() - 24h` OR connection status is not 'active'
- Tooltip / tap: shows exact last sync time

### Data Dependencies
- `accounts` (connection status, last_synced_at)
- `account_connection_log` (sync history, error log)
- Basiq API / NZ connector API

---

## 2. Net Position (Hero Card)

### What It Is
The single headline figure on the Accounts tab — the user's total net financial position across all linked accounts. Shown as a large hero number at the top of the screen.

### Formula
```
net_position = total_assets − total_liabilities

total_assets =
    sum(accounts.balance WHERE account_type IN ('transaction', 'savings', 'term_deposit')
                           AND include_in_net_worth = true
                           AND is_active = true)

total_liabilities =
    sum(ABS(accounts.balance) WHERE account_type IN ('credit_card', 'loan', 'mortgage')
                                 AND include_in_net_worth = true
                                 AND is_active = true)
    // credit_card and loan balances stored as negative integers; ABS for display
```

### Alex's Mock Values
```
ANZ Everyday (transaction):  $1,847  → asset
ANZ NetSaver (savings):     $12,450  → asset
ANZ Credit Visa (credit):    -$843   → liability ($843 debt)

total_assets      = $14,297
total_liabilities =    $843
net_position      = +$13,454
```

### Display Labels
```
IF net_position > 0: "+$X,XXX" (brand purple or green)
IF net_position = 0: "$0" (neutral)
IF net_position < 0: "-$X,XXX" (red)
```

### Sub-line breakdown
```
"$14,297 assets · $843 debt"
```
Always shown below the hero number so users understand the composition.

### Edge Cases
| Scenario | Handling |
|---|---|
| No accounts linked | Show empty state: "Connect your first bank account" |
| Account sync failing (stale balance) | Show balance with "⚠ Last updated X hours ago" |
| Investment/super accounts | Excluded by default; shown separately if `include_in_net_worth = true` |
| Mortgage offset | Treated as savings (offsets loan balance); type = 'savings', flagged as offset |

### Data Dependencies
- `accounts` (balance, account_type, include_in_net_worth, is_active)

### Calculation Run Schedule
- On every Basiq webhook (balance change)
- On app open
- On manual refresh

---

## 3. Account Detail View

### What It Is
A bottom sheet that appears when the user taps any account row. Shows account metadata and the last 5 transactions for that account.

### Content
```
Sheet header:
  - Institution name + account name
  - Masked account number (last 4 digits)
  - Current balance (large)
  - Account type label

Transaction list (last 5, sorted by date DESC):
  - Merchant name
  - Date
  - Amount (debit in red, credit in green)
  - Category icon
  - "See all transactions" link → navigates to full transaction list (future feature)
```

### Interest Rate Display (Savings Accounts)
If `account_type = 'savings'` AND `interest_rate IS NOT NULL`:
- Show interest rate badge next to balance: "3.75% p.a."
- Show projected monthly interest: `balance × (interest_rate / 12)` formatted as "$XX/month"

### Data Dependencies
- `accounts` (all fields)
- `transactions` (last 5 for account, ordered by date DESC)
- `categories` (for category icons)

---

## 4. Transaction Review Queue (To Review Tab)

### What It Is
A queue of recently imported transactions where the AI has suggested a category. The user confirms or corrects the suggestion. This is the primary data quality loop — the more transactions reviewed, the more accurate Totem's categorisation becomes for that user.

### What Qualifies for the Queue
```
SELECT transactions WHERE user_id = current_user
  AND review_status = 'pending'
  AND is_income = false
  AND is_transfer = false
  AND source IN ('bank_feed', 'csv_import')
ORDER BY date DESC, amount DESC
```

Rules:
- Only non-income, non-transfer transactions enter the queue
- Internal account transfers (detected by matching debit/credit amounts between linked accounts on the same day) are auto-excluded
- Transactions already reviewed (review_status = 'approved' or 'corrected') are removed from queue

### AI Categorisation Confidence
Every AI-assigned category gets a confidence score:
```
confidence_score: 0.00–1.00

Display mapping:
  >= 0.80:  badge = "High"   (green)   — AI is very confident, quick confirm expected
  >= 0.55:  badge = "Medium" (amber)   — AI is reasonably confident, worth a glance
  < 0.55:   badge = "Low"    (red)     — shown in queue but flagged for closer review
             // Low confidence items are shown last in queue ordering
```

### Queue Ordering
```
1. High confidence items (sorted by date DESC) — fastest to confirm
2. Medium confidence items (sorted by date DESC)
3. Low confidence items (sorted by date DESC)
// Within each confidence tier: largest amounts first — higher stakes decisions up front
```

### Approve Flow (✓ button)
```
1. User taps ✓ on a review item
2. Item collapses with CSS max-height transition (300ms ease-out)
3. transaction_reviews record updated: review_status = 'approved', reviewed_at = NOW()
4. acctReviewCount increments by 1
5. Progress bar re-renders
6. If acctReviewCount reaches 30: milestone event fires (see Section 5)
7. Queue re-sorts (next item animates into position)
```

### Reject/Reclassify Flow (✗ button)
```
1. User taps ✗ on a review item
2. Category picker bottom sheet opens (acct-cat-sheet)
3. 11 category options displayed in 3-column grid
4. User selects correct category
5. Sheet closes
6. transaction.category_id updated to selected category
7. transaction.category_assigned_by updated to 'user'
8. review_status = 'corrected'
9. Item collapses (same animation as approve)
10. acctReviewCount increments by 1
```

### Category Picker Categories (11 options, fixed)
| Category | Emoji | Slug |
|---|---|---|
| Eating Out | 🍔 | eating-out |
| Groceries | 🛒 | groceries |
| Transport | 🚗 | transport |
| Entertainment | 🎭 | entertainment |
| Shopping | 🛍️ | shopping |
| Health | 💊 | health |
| Utilities | 💡 | utilities |
| Subscriptions | 📱 | subscriptions |
| Personal Care | 💅 | personal-care |
| Insurance | 🛡️ | insurance |
| Other | 📦 | other |

### Learning Loop
```
When a user corrects a category (review_status = 'corrected'):
1. Log the correction: original AI category vs user-selected category for this merchant
2. If this merchant has been corrected to the same category 2+ times:
   → Auto-generate an AI categorisation rule: merchant_name → user's preferred category
   → Add to categorisation_rules with created_by = 'ai_suggestion'
   → This rule auto-applies to all future transactions from this merchant
```

### Empty State
When all items in queue are reviewed:
- Show a celebration state: "All caught up! ✓"
- Show the milestone progress (e.g. "24/30 transactions reviewed")
- If milestone reached (30/30): show green milestone banner with AI features unlocked message

### Data Dependencies
- `transactions` (pending reviews)
- `transaction_reviews` (review status, timestamps)
- `categories` (for category picker)
- `categorisation_rules` (for auto-rule generation on corrections)

### Calculation Run Schedule
- Queue refreshes on every Basiq webhook (new transactions enter queue immediately)
- acctReviewCount recalculated on app open (count WHERE review_status IN ('approved', 'corrected'))

---

## 5. 30-Transaction AI Activation Gate

### What It Is
A gamified progress milestone. Totem's AI features (goal suggestions, proactive insights, advanced categorisation) activate after the user has reviewed 30 transactions. This ensures sufficient data quality before AI features make recommendations. Displayed as a progress bar in the To Review tab.

### Formula
```
reviewed_count = count(transaction_reviews WHERE user_id = current_user
                                            AND review_status IN ('approved', 'corrected'))

progress_pct = (reviewed_count / 30) × 100   // cap at 100
```

### Display States
```
IF reviewed_count < 30:
  - Progress bar at (reviewed_count / 30)%
  - Label: "X of 30 transactions reviewed"
  - Sub-label: "Review transactions to unlock AI features"
  - Milestone button: not yet active (greyed)

IF reviewed_count >= 30:
  - Progress bar: full green (100%)
  - Label: "30/30 — AI features unlocked! ✓"
  - Banner: brand-coloured celebration state
  - user_ai_activation.activated_at = NOW() (written once)
```

### What Unlocks at 30
- AI Goal Suggestions (Plan screen → Goals tab → AI suggestion cards appear)
- Proactive AI Insights (Dashboard AI Insight card becomes more personalised)
- Advanced categorisation model — switches from generic model to user-personalised model
- Spending Habit goal baseline computation activates

### Progress Bar Spec
```
Track: full width of card, height 6px, background rgba(255,255,255,0.08)
Fill:  brand purple up to 29/30; switches to green at 30/30
Milestone markers: none (single threshold milestone, not incremental)
Animation: fill transitions width on each approval (0.4s ease-out)
```

### Data Dependencies
- `transaction_reviews` (reviewed count)
- `user_ai_activation` (from Plan schema — tracks milestone reached_at)

---

## 6. Categorisation Rule Engine (Rules Tab)

### What It Is
A list of active rules that automatically assign categories to future transactions, eliminating the need to manually review recurring merchants. Rules can be created by the user or suggested by AI.

### Rule Types
| Match Type | Description | Example |
|---|---|---|
| `merchant_name` | Exact match on normalised merchant name | "NETFLIX" → Subscriptions |
| `description_contains` | Substring match on raw bank description | "AFTERPAY" → Shopping |
| `amount_range` | Amount falls within a defined range (combined with merchant) | $X–$Y at merchant Y → Insurance |

### Rule Sources
| Source | Badge | How Created |
|---|---|---|
| `user` | "You" badge (muted/brand) | User manually creates via "Add Rule" flow |
| `ai_suggestion` | "✨ AI" badge (brand) | AI auto-generates after 2+ consistent corrections to same category for same merchant |

### Rule Priority / Precedence
```
When a new transaction arrives, rules apply in this order:
1. User-created rules (highest priority — user intent overrides AI)
2. AI-suggested rules (active)
3. AI categorisation model (fallback — runs if no rule matches)

Within user rules: most recently created rule wins on conflict
Within AI rules: most recently suggested rule wins on conflict
```

### Rule Application Flow
```
On new transaction insert:
  FOR each active rule (ORDER BY created_by = 'user' DESC, created_at DESC):
    IF rule.match_type = 'merchant_name' AND transaction.merchant_name_normalised ILIKE rule.match_value:
      transaction.category_id = rule.category_id
      transaction.category_assigned_by = 'rule'
      transaction.category_confidence = 1.00
      BREAK  // first matching rule wins
    IF rule.match_type = 'description_contains' AND transaction.description ILIKE '%' || rule.match_value || '%':
      // same assignment
      BREAK
    IF rule.match_type = 'amount_range':
      // check both merchant AND amount bounds
      BREAK
  IF no rule matched:
    → Send to AI categorisation model
```

### ON/OFF Toggle Behaviour
- Toggle changes `categorisation_rules.is_active` between true/false
- Disabling a rule does NOT re-categorise historical transactions already classified by it
- Re-enabling a rule does NOT retroactively apply to historical transactions
- Only affects new transactions going forward

### Edit Rule
- Each rule row stores `data-merchant`, `data-cat`, `data-emoji` attributes
- Tapping any rule row opens `acct-rule-edit-sheet` (bottom sheet with merchant context + 11-category picker)
- ON/OFF toggle and trash icon have `event.stopPropagation()` to prevent triggering row click
- On save: row icon, category label, and `data-cat` update in-place; if the rule was AI-sourced, its badge downgrades from "✨ AI" to "You" (user has overridden the AI suggestion)
- Backend: `UPDATE categorisation_rules SET category_id = $new, updated_by = 'user' WHERE id = $rule_id`
- Add Rule flow uses separate `acct-add-rule-sheet` with `add-rule-cat-grid` ID and `pickAddRuleCat()` function to avoid conflict with the edit sheet's `rule-cat-grid` and `pickRuleCat()`

### Delete Rule
- Hard delete from `categorisation_rules`
- Historical transactions categorised by this rule retain their category (soft effect — only future is affected)
- If deleting an AI rule that was auto-generated, no new rule is auto-generated until 2 more corrections occur

### Alex's Mock Rules (8 total)
| # | Merchant | → Category | Source |
|---|---|---|---|
| 1 | NETFLIX | Subscriptions | AI |
| 2 | SPOTIFY | Subscriptions | AI |
| 3 | WOOLWORTHS | Groceries | AI |
| 4 | TRANSLINK / OPAL | Transport | AI |
| 5 | UBER EATS | Eating Out | You |
| 6 | AMAZON | Shopping | You |
| 7 | CHEMIST WAREHOUSE | Health | You |
| 8 | GYM / FITNESS FIRST | Personal Care | You |

### Data Dependencies
- `categorisation_rules` (from Dashboard/database-schema.md — same table, read here)
- `transactions` (rule application target)
- `categories` (category assignment)

### Calculation Run Schedule
- Rules evaluated on every new transaction (synchronous, before AI categorisation fallback)
- Rule list re-fetched on app open and after any rule add/edit/delete

---

## 7. Subscriptions Tab

### What It Is
A dedicated tab listing all detected recurring subscriptions for the user. Surfaces subscriptions that have been auto-detected by the recurring bill detection engine (confirmed status, `subscription_flag = true`). Accessed via the Subs tab in the Accounts tab bar, or directly from the Dashboard "Manage subscriptions" button.

### Entry Point from Dashboard
```
Dashboard Tier 2 — Subscription Load card → "Manage" button
→ calls navigateToAcctSubs()
→ navigate('accounts') — screen transition
→ setTimeout(300ms) → switchAcctTab('subs')
// Delayed to let the screen transition complete before switching tab
```

### Subscription List
Each row displays:
- `.list-icon` rounded-square icon (service-brand-tinted background, 36×36px, border-radius 10px)
- Merchant / service name (14px, weight 600)
- Category tag (11px, var(--text2)) — always "Subscriptions" for this tab
- Monthly cost (right-aligned, 13px, weight 600)
- Frequency badge (e.g. "Monthly", "Annual") where known

### Alex's Mock Subscriptions (11 active)
| Service | Monthly Cost | Icon bg | Emoji |
|---|---|---|---|
| Netflix | $18.99 | `rgba(229,9,20,0.12)` | 🎬 |
| Spotify | $12.99 | `rgba(30,215,96,0.12)` | 🎵 |
| Apple TV+ | $12.99 | `rgba(0,125,250,0.12)` | 📺 |
| YouTube Premium | $22.99 | `rgba(255,0,0,0.12)` | 📹 |
| Disney+ | $13.99 | `rgba(17,60,207,0.12)` | 🏰 |
| iCloud+ | $4.99 | `rgba(0,125,250,0.12)` | ☁️ |
| Amazon Prime | $9.99 | `rgba(255,153,0,0.12)` | 🛍️ |
| ChatGPT Plus | $29.99 | `rgba(16,163,127,0.12)` | 🤖 |
| F45 Training | $65.00 | `rgba(124,110,250,0.12)` | 🏋️ |
| Adobe CC | $89.99 | `rgba(255,0,0,0.12)` | 🎨 |
| Headspace | $12.99 | `rgba(255,161,0,0.12)` | 🧘 |

**Total monthly commitment:** ~$294.91

### Header Behaviour
- Tab header action label: "11 active" (static text, no onclick)
- `updateAcctHeader('subs')` sets action text to subscription count

### Data Dependencies
- `recurring_bills` (WHERE `subscription_flag = true` AND `status = 'confirmed'`)
- `transactions` (for last charge date, to validate subscription is still active)
- `categories` (Subscriptions category)

### Calculation Run Schedule
- Subscription list re-fetched on app open and after any recurring bill status change
- Count shown in header updates whenever recurring bills table changes

---

## Appendix: Calculation Run Schedule

| Feature | Trigger |
|---|---|
| Net Position | Every Basiq webhook + app open + manual refresh |
| Account Detail | On demand (tap account row) |
| To Review Queue | Every Basiq webhook + app open |
| Review Count / Progress | Every review action (approve or correct) |
| AI Activation Milestone | On review action when count reaches 30 |
| Rule Application | On every new transaction insert (synchronous) |
| Rule List | App open + any rule change |
| Subscription List | App open + any recurring bill status change |
| Live Feed Indicator | Continuous (rechecks last_synced_at every 60s while screen active) |
