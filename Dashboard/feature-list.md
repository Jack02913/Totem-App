# Totem — Dashboard Feature List
## Math, Algorithms & Data Dependencies

> **Scope:** Dashboard page only. All values are per-user. All currency in AUD (NZ flow identical with NZD). All averages are rolling unless noted.

---

## 1. Safe to Spend

### What It Is
The single most important number on the dashboard. The amount of money the user can freely spend between now and their next payday, after all committed outgoings have been accounted for. Scoped to the current pay period.

### Formula
```
Safe to Spend =
    Net income received this pay period
  − Total discretionary spend this pay period (already occurred)
  − Confirmed recurring bills due BEFORE next payday (not yet charged)
  − Scheduled savings auto-transfers due BEFORE next payday (not yet transferred)
```

### Calculation Detail
```
safe_to_spend = pay_period.actual_income
              - sum(transactions WHERE pay_period_id = current
                                 AND is_income = false
                                 AND is_transfer = false
                                 AND is_recurring = false)
              - sum(recurring_bills WHERE next_due_date <= pay_period.end_date
                                    AND last_charged_date < pay_period.start_date
                                    AND status = 'confirmed')
              - sum(savings_goals WHERE next_transfer_date <= pay_period.end_date
                                   AND auto_transfer_enabled = true)
```

### Edge Cases
| Scenario | Handling |
|---|---|
| Income not yet received (payday today) | Use `pay_period.expected_income` until actual deposit clears |
| Negative result | Show $0 with a red banner: "You've overspent this pay period" |
| Multiple income sources in one period | Sum all confirmed income deposits from the period |
| Irregular income user | Switch formula — see Section 1a below |

### 1a. Irregular Income Fallback
```
Safe to Spend (irregular) =
    Liquid account balance (all linked transaction accounts)
  − Confirmed bills due in next 30 days
  − Savings target for next 30 days
```

### Data Dependencies
- `pay_periods` (current period start/end, expected/actual income)
- `transactions` (this period's discretionary spend)
- `recurring_bills` (confirmed, due before payday)
- `savings_goals` (auto-transfer schedule)
- `accounts` (balance, for irregular fallback)

---

## 2. Month End Projection

### What It Is
A forward-looking estimate of the user's net surplus or deficit by the end of the current calendar month. Answers: "If I keep spending at this pace, where will I be on the last day of this month?"

### Formula
```
Month End Projection =
    Total expected income this calendar month
  − Total confirmed recurring expenses this calendar month
  − Projected total discretionary spend this calendar month
```

### Projected Discretionary Spend
```
daily_burn_rate = actual_discretionary_spend_mtd / days_elapsed_this_month

projected_discretionary = daily_burn_rate × total_days_in_month
```

### Full Calculation
```
projected_surplus =
    (sum of expected income deposits in current calendar month)
  - (sum of confirmed recurring bills in current calendar month)
  - ((actual_discretionary_mtd / days_elapsed) × days_in_month)
```

### Status Labels
| Condition | Label | Colour |
|---|---|---|
| `projected_surplus > 0` | "On Track · +$X projected" | Green |
| `projected_surplus < 0 AND > -200` | "At Risk · -$X projected" | Amber |
| `projected_surplus < -200` | "Overspend Risk · -$X projected" | Red |

### Edge Cases
| Scenario | Handling |
|---|---|
| First 3 days of month | Use prior month's daily average instead of current (insufficient data) |
| Irregular spend event (e.g. annual insurance) | Exclude one-off amounts > 3× the category's daily average from the burn rate calculation |
| Fortnightly pay straddles months | Include full fortnightly deposit expected in this calendar month |

### Data Dependencies
- `transactions` (actual_discretionary_mtd)
- `recurring_bills` (confirmed bills in month)
- `pay_periods` + `users.pay_cycle_*` (expected income in month)
- System date for `days_elapsed` and `days_in_month`

---

## 3. Payday Anchor

### What It Is
A persistent pill at the top of Tier 1 showing: days until next payday, the employer/source name, the expected deposit amount, and where in the current pay cycle the user sits.

### Detection Algorithm
```
1. Fetch all transactions WHERE is_income = true for last 6 months
2. Group by (merchant_name, amount_bucket)  // amount_bucket = round to nearest $50
3. For each group, calculate gaps between consecutive transactions (in days)
4. If stddev(gaps) < 3 days AND count >= 3:
     → Flag as regular pay cycle
     → cycle_length = mode(gaps)  // most common gap
     → pay_source = merchant_name
     → expected_amount = median(amounts)
5. Score confidence:
     - HIGH:   count >= 4 AND stddev < 2 AND amount_variance < 5%
     - MEDIUM: count >= 3 AND stddev < 4 AND amount_variance < 10%
     - LOW:    count == 2 OR stddev >= 4
6. If confidence HIGH or MEDIUM → auto-set, prompt user to confirm
7. If confidence LOW OR no pattern found → prompt user to set manually
```

### Pay Cycle Progress
```
days_elapsed = today - pay_period.start_date
days_remaining = pay_period.end_date - today
progress_pct = (days_elapsed / pay_period.length_days) × 100
```

### Display Logic
```
IF days_remaining == 0: "Payday today!"
IF days_remaining == 1: "Payday tomorrow"
IF days_remaining <= 7: "Payday in X days"
IF irregular_income:    "Next bill in X days"  // show closest upcoming bill instead
```

### Data Dependencies
- `transactions` (income detection)
- `pay_periods` (current period record)
- `users` (pay_cycle_type, pay_cycle_day, income_source_name)

---

## 4. AI Insight Card

### What It Is
A single, high-priority, actionable insight shown on Tier 1. Only one insight shown at a time. The most important one wins.

### Insight Types & Base Scores

| Type | Severity | Urgency | Dollar Weight | Example |
|---|---|---|---|---|
| Category over budget | 3 | 2 | overage amount | "Eating out $87 over budget" |
| Category at risk (on pace to overspend) | 2 | 2 | projected overage | "Groceries on pace to overspend by $40" |
| Unused subscription | 2 | 1 | monthly cost | "Haven't used Binge in 47 days ($18/mo)" |
| Unusual transaction | 2 | 3 | transaction amount | "Unusual $340 charge at [Merchant]" |
| Payday approaching, low balance | 3 | 3 | shortfall amount | "You have $45 left with 3 days until payday" |
| Safety net dropped below target | 2 | 1 | months below target | "Safety net dropped to 1.8 months" |
| Savings goal at risk | 2 | 2 | shortfall to goal | "Behind on House Deposit goal by $200" |
| Positive: savings rate improved | 0 | 0 | improvement amount | "Savings rate up 4% this month 🎉" |
| Positive: category under budget | 0 | 0 | savings amount | "You're $60 under on eating out this month" |

### Priority Score Formula
```
priority_score =
    (severity × 0.35)
  + (urgency × 0.30)
  + (dollar_impact_score × 0.25)
  + (novelty × 0.10)

dollar_impact_score:
  > $200  → 3
  > $50   → 2
  > $20   → 1
  otherwise → 0

novelty:
  insight NOT shown to user in last 7 days → 1
  otherwise → 0
```

### Selection Logic
```
1. Generate all currently-true insights for user
2. Score each with the formula above
3. SELECT TOP 1 ORDER BY priority_score DESC
4. If user dismisses → mark dismissed, re-run selection excluding dismissed insights for 24h
5. If user taps "See breakdown" → mark acted_on, log action type
```

### Insight Evaluation (runs on)
- Every Basiq webhook (new transaction)
- Every time user opens the app (check if new top insight differs from shown insight)
- Daily at 6am (proactive checks)

### Data Dependencies
- `transactions` (for unusual transaction, category budgets)
- `budgets` (for over-budget, at-risk)
- `recurring_bills` (for unused subscription check — cross-reference with transaction activity in last 30 days per merchant)
- `savings_goals` (for goal risk)
- `accounts` (for low balance)
- `ai_insights` (for novelty check — last shown_at per type)
- `lifestyle_creep_snapshots` (for creep alerts)

---

## 5. Upcoming Bills

### What It Is
A list of confirmed recurring charges due in the next 7–14 days, with merchant name, amount, due date, and urgency label. Fed by confirmed recurring bills only.

### Recurring Bill Detection Algorithm
```
Step 1: Candidate identification
  - Fetch all transactions for user, last 90 days
  - Group by merchant_name (normalised — strip trailing numbers, "PTY LTD" etc.)
  - For each group: calculate inter-transaction intervals (days between consecutive)

Step 2: Pattern scoring
  FOR each merchant group:
    intervals = [days between consecutive transactions]
    mean_interval = avg(intervals)
    stddev_interval = stddev(intervals)
    amount_variance = stddev(amounts) / mean(amounts)  // coefficient of variation

    // Check for common period buckets
    IF 6 <= mean_interval <= 8:    period = 'weekly'
    IF 13 <= mean_interval <= 16:  period = 'fortnightly'
    IF 27 <= mean_interval <= 34:  period = 'monthly'
    IF 85 <= mean_interval <= 100: period = 'quarterly'
    IF 355 <= mean_interval <= 380: period = 'annual'
    ELSE: skip (no clear pattern)

    confidence = 'high'   IF count >= 3 AND stddev < 3 AND amount_variance < 0.05
    confidence = 'medium' IF count >= 2 AND stddev < 5 AND amount_variance < 0.15
    confidence = 'low'    IF count == 2 AND stddev >= 5

Step 3: Next due date projection
    next_due = last_transaction_date + mean_interval (rounded to nearest day)

Step 4: User approval queue
    - confidence 'high':   show in approval queue with pre-ticked checkbox
    - confidence 'medium': show in approval queue with unticked checkbox
    - confidence 'low':    do not show unless user enables "show uncertain"
    - status = 'pending_approval' until user confirms or dismisses
```

### Display Window
```
show_bills WHERE next_due_date BETWEEN today AND today + 14 days
             AND status = 'confirmed'
ORDER BY next_due_date ASC
LIMIT 5 on dashboard (show "See all" for full list)
```

### Urgency Labels
```
IF next_due_date == today:          "Today"         (red)
IF next_due_date == today + 1:      "Tomorrow"      (amber)
IF next_due_date <= today + 3:      "In X days"     (amber)
IF next_due_date <= today + 7:      "In X days"     (text-muted)
IF next_due_date > today + 7:       "X days away"   (text-muted)
```

### Data Dependencies
- `transactions` (pattern detection)
- `recurring_bills` (confirmed list, next_due_date)

---

## 6. Budget Burn Rate

### What It Is
Per-category progress bars showing how much of a user-set budget has been spent, colour-coded by whether the spend rate is on track, at risk, or over budget relative to where they should be in the period.

### Formula
```
// What should have been spent by today if spending evenly
expected_at_this_point = (days_elapsed / period_length_days) × budget_amount

// Actual vs expected
burn_pct       = (actual_spend / budget_amount) × 100
expected_pct   = (expected_at_this_point / budget_amount) × 100

// Overspend projection
projected_end  = (actual_spend / days_elapsed) × period_length_days
projected_over = projected_end - budget_amount
```

### Status Logic
```
IF actual_spend > budget_amount:
    status = 'OVER'
    colour = red
    label  = "Over Budget · $X over"

ELSE IF actual_spend > (expected_at_this_point × 1.10):
    status = 'AT_RISK'
    colour = amber
    label  = "At Risk · on pace to overspend by $X"

ELSE:
    status = 'ON_TRACK'
    colour = green
    label  = "On Track"
```

### Period Marker (key UX detail)
The progress bar shows a faint vertical tick at `expected_pct` position. This lets users see at a glance whether their fill is ahead or behind the expected pace. Without this, 75% used means nothing without knowing it's day 10 of 14 vs day 3 of 14.

### Display Rules
- Only show categories where user HAS set a budget
- Do not show categories with no budget set (they appear on the full categories page only)
- If no budgets set: show a prompt card — "Set spending budgets to track burn rate"
- Show maximum 5 categories on dashboard; "See all budgets" link to full list

### Data Dependencies
- `budgets` (budget_amount, period_type)
- `transactions` (actual_spend per category in current period)
- `pay_periods` (days_elapsed, period_length_days)
- `categories` (category names, icons)

---

## 7. Subscription Load

### What It Is
A summary card showing how many active recurring subscriptions the user has, the total monthly cost, and the annualised cost. Pulls from confirmed recurring bills filtered to subscriptions (excludes rent, utilities — see categorisation note).

### Subscription Identification
Recurring bills are tagged as subscriptions if:
- Their merchant MCC code maps to entertainment/streaming/software (MCCs: 5815, 5816, 7372, etc.)
- OR the user has manually categorised the transaction as a subscription
- AND the transaction amount is < $200/month (above this is more likely a utility or insurance)

### Calculations
```
active_subscriptions = recurring_bills WHERE status = 'confirmed'
                                        AND category = 'subscription'
                                        AND last_charged_date >= today - 60 days

monthly_cost = sum(
    CASE frequency
        WHEN 'weekly'     THEN amount × 52 / 12
        WHEN 'fortnightly' THEN amount × 26 / 12
        WHEN 'monthly'    THEN amount
        WHEN 'quarterly'  THEN amount / 3
        WHEN 'annual'     THEN amount / 12
    END
) FOR each active_subscription

annualised_cost = monthly_cost × 12

subscription_count = count(active_subscriptions)
```

### Change Tracking
```
subscriptions_this_year = count WHERE first_detected_date >= current_year start
prev_year_count = count WHERE first_detected_date < current_year start AND last_charged >= prev_year start

change_label:
  IF subscriptions_this_year > 0: "+X this year"  (amber)
  IF subscriptions_this_year < 0: "-X this year"  (green)
  ELSE: "Unchanged"                                (neutral)
```

### Unused Subscription Detection (feeds into AI Insight)
```
FOR each active_subscription:
  last_transaction_matching_merchant = MAX(transaction.date) WHERE merchant = subscription.merchant_name
  days_since_used = today - last_transaction_matching_merchant
  // Note: streaming services often can't be detected by transaction activity alone
  // For services where activity CAN be detected, flag if days_since_used > 30
```

### Data Dependencies
- `recurring_bills` (confirmed subscriptions)
- `transactions` (last activity per merchant)

---

## 8. Safety Net

### What It Is
The number of months the user could sustain their current lifestyle if their income stopped today. Calculated from liquid savings divided by average monthly expenses.

### Formula
```
liquid_savings = sum(accounts.balance WHERE account_type IN ('savings', 'transaction'))
                 - committed_expenses_next_month  // don't count money that's already spoken for

avg_monthly_expenses = sum(transactions.amount WHERE is_income = false
                                                AND is_transfer = false
                                                AND date >= today - 90 days)
                       / 3  // 3-month rolling average

safety_net_months = liquid_savings / avg_monthly_expenses
```

### Display
```
IF safety_net_months < 1:    colour = red,   label = "Critical — less than 1 month"
IF safety_net_months < 3:    colour = amber, label = "Below 3-month target"
IF safety_net_months >= 3:   colour = green, label = "Healthy"
IF safety_net_months >= 6:   colour = brand, label = "Excellent — 6+ months"
```

### Notes
- Use 3-month rolling average for monthly expenses (not a single month — avoids seasonal distortions)
- Exclude investment accounts and super from liquid savings (not liquid)
- Exclude mortgage offset account balance if it is committed to a home loan
- Recalculate daily (balance changes) or on any new transaction

### Data Dependencies
- `accounts` (balances, account types)
- `transactions` (3-month expense history)
- `recurring_bills` (committed next month expenses)

---

## 9. Savings Velocity

### What It Is
The percentage of each paycheck the user saves on average. Tracks direction (improving/declining) over time.

### Formula
```
// Per pay period
period_savings_rate = (
    actual_income_this_period
  - total_spend_this_period  // includes committed and discretionary
) / actual_income_this_period × 100

// Rolling 3-month average (smooths irregular months)
avg_savings_rate = avg(period_savings_rate) for last 3 completed pay periods
```

### Direction Tracking
```
previous_3mo_avg = avg(period_savings_rate) for periods 4–6 months ago
direction_delta  = avg_savings_rate - previous_3mo_avg
direction_label:
  IF delta > 1%:   "↑ Up from X%"    (green)
  IF delta < -1%:  "↓ Down from X%"  (red)
  ELSE:            "Stable"           (neutral)
```

### Notes
- Requires minimum 1 completed pay period to show (not current in-progress period)
- For the current in-progress period, show a projected rate: `projected = (income - projected_spend) / income × 100`
- Transfers to savings accounts DO count as "savings" in this calculation

### Data Dependencies
- `pay_periods` (all completed periods)
- `transactions` (income and spend per period)
- `savings_goals` (savings transfers, used to cross-validate)

---

## 10. Lifestyle Creep

### What It Is
A metric that detects whether the user's lifestyle expenses are growing disproportionately relative to their income growth. Compares 3-month rolling average spend to 12-month rolling average spend, and cross-references with income trend.

### Minimum Data Requirements
| State | Condition | Display |
|---|---|---|
| Building | < 3 months of data | Progress bar showing months complete |
| Partial | 3–11 months of data | Lifestyle Creep shows, labeled "based on X months" |
| Full | 12+ months of data | Full 3-month vs 12-month comparison with income overlay |

### Formulas
```
// 3-month spending average
avg_spend_3mo = sum(transactions.amount WHERE is_income = false
                                          AND is_transfer = false
                                          AND date >= today - 90 days)
                / 3

// 12-month spending average
avg_spend_12mo = sum(transactions.amount WHERE is_income = false
                                           AND is_transfer = false
                                           AND date >= today - 365 days)
                 / 12

// 3-month income average
avg_income_3mo = sum(transactions.amount WHERE is_income = true
                                           AND date >= today - 90 days)
                 / 3

// 12-month income average
avg_income_12mo = sum(transactions.amount WHERE is_income = true
                                            AND date >= today - 365 days)
                  / 12

// Lifestyle creep ratio
spend_growth  = (avg_spend_3mo - avg_spend_12mo) / avg_spend_12mo × 100
income_growth = (avg_income_3mo - avg_income_12mo) / avg_income_12mo × 100

creep_delta = spend_growth - income_growth
```

### Interpretation
```
IF creep_delta > 10%:   "Lifestyle Creep Detected" (red)    — spending growing faster than income
IF creep_delta 5–10%:   "Watch This" (amber)                — mild creep, monitor
IF creep_delta -5–5%:   "Balanced" (green)                  — spend and income growing in line
IF creep_delta < -5%:   "Improving" (brand)                 — spending growing slower than income (positive)
```

### CSV Import Handling
- When user imports CSV history on setup, use imported transactions to satisfy the 3-month and 12-month data requirements immediately
- Mark imported transactions with `source = 'csv_import'` for audit trail
- If CSV covers < 3 months: still shows building state
- If CSV covers 3–11 months: partial state, labeled accordingly
- If CSV covers 12+ months: full metric available from day one

### Data Dependencies
- `transactions` (12-month history, income and expense)
- `lifestyle_creep_snapshots` (monthly snapshots for historical chart)
- `users` (csv_import_start_date, for data age tracking)

---

## 11. Spend as % of Income (Category Breakdown)

### What It Is
Top spending categories displayed as a percentage of monthly income, rather than percentage of total spend. Allows users to benchmark categories against their actual earnings.

### Formula
```
category_pct_of_income = (category_spend_this_month / monthly_income) × 100

monthly_income = sum(transactions WHERE is_income = true
                                   AND date >= first_day_of_current_month)
                 // OR expected_monthly_income if month not complete
```

### Display Logic
- Show top 5 categories by absolute spend value
- Sort: highest spend first
- Apply colour coding to the bar:
  - Default: brand colour
  - Eating Out / Takeaway if over-budget: red bar
  - Any category > 30% of income: amber bar (high proportion flag)
- Show "See all categories" to navigate to full breakdown

### Data Dependencies
- `transactions` (current month, categorised)
- `categories` (names, icons)
- `pay_periods` or `users` (monthly income figure)

---

## Appendix: Calculation Run Schedule

| Feature | Trigger |
|---|---|
| Safe to Spend | On every new transaction (Basiq webhook) + app open |
| Month End Projection | On every new transaction + app open |
| Payday Anchor | On pay detected (Basiq webhook) + daily at 00:01 |
| AI Insight | On every new transaction + daily at 06:00 |
| Upcoming Bills | On bill detection/confirmation + daily at 06:00 |
| Budget Burn Rate | On every new transaction + app open |
| Subscription Load | On new recurring bill detection/confirmation |
| Safety Net | On every new transaction + daily balance refresh |
| Savings Velocity | On pay period close + app open |
| Lifestyle Creep | Monthly snapshot (1st of each month at 02:00) |
| Category % of Income | On every new transaction + app open |
