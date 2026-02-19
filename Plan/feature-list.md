# Totem — Plan Screen Feature List
## Goals + Budgets: Math, Algorithms & Data Dependencies

> **Scope:** Plan screen only (Tab 2). Two sub-tabs: Goals and Budgets.
> **Note:** The `savings_goals` table from `Dashboard/database-schema.md` is superseded by the comprehensive goals schema in `Plan/database-schema.md`.

---

## Goal Types Overview

Three distinct goal types, each with different tracking logic, visual treatment, and notification behaviour:

| Type | Purpose | Resets? | Tracks |
|---|---|---|---|
| **Save Up** | Accumulate money toward a target | No | Balance growth |
| **Spending Habit** | Reduce spending in a category by X% | Monthly (auto-renew) | Category spend vs. target |
| **Debt Paydown** | Eliminate a credit card or loan balance | No | Liability reduction |

---

## 1. Save Up Goal

### What It Is
A savings accumulation goal with a target amount, optional target date, and optional monthly contribution. Tracks how much has been saved toward the target in a linked account.

### Formulas

```
progress_pct = (current_amount / target_amount) × 100

// Monthly contribution required to hit target on time
required_monthly = (target_amount - current_amount) / months_until_target_date

// Average monthly contribution (rolling 3-month)
avg_monthly = sum(contributions in last 3 months) / 3

// Projected completion at current rate
projected_completion_months = (target_amount - current_amount) / avg_monthly
projected_completion_date = today + projected_completion_months (in months)
```

### Status Logic
```
IF projected_completion_date <= target_date - 30 days:  status = 'AHEAD'
IF projected_completion_date <= target_date:             status = 'ON_TRACK'
IF projected_completion_date <= target_date + 30 days:  status = 'AT_RISK'
IF projected_completion_date > target_date + 30 days:   status = 'BEHIND'
IF no target_date set:                                   status = 'IN_PROGRESS' (no deadline pressure)
```

### Progress Tracking
- Current amount is read from the `goal_save_up.current_amount` field
- Updated either by:
  1. Auto-detected transfer to the linked savings account (Basiq webhook)
  2. Manual "Add to Goal" button tap (user-entered amount)
  3. Auto-transfer (if `auto_transfer_enabled = true`) — same mechanism as savings_goals auto-transfer in Dashboard schema

### AI-Generated Save Up Goals
Trigger conditions (background job, runs daily):
```
IF avg_monthly_surplus (last 3 months) > $200 AND user has no active Save Up goals:
  → Generate suggestion: "You have ~$X surplus/month. Want to put it toward something?"

IF user mentions a savings target in AI Chat (keyword detection):
  → Immediately generate goal preview in chat thread
```

### Edge Cases
| Scenario | Handling |
|---|---|
| No target date set | Show "In Progress" — no deadline pressure status |
| Current amount > target amount | Goal complete — trigger celebration flow |
| Linked account balance drops below current_amount | Recalculate — user may have spent from savings |
| avg_monthly_contribution = 0 | Show "No contributions yet" instead of projected date |

### Data Dependencies
- `goals` (base goal record)
- `goal_save_up` (target, current, contribution)
- `accounts` (linked savings account balance)
- `transactions` (monthly contribution history — transfers to linked account)

---

## 2. Spending Habit Goal

### What It Is
Totem's signature goal type. Not available in any other AU/NZ finance app. Tracks spending in a specific category against a user-set or AI-calculated monthly target. Auto-renews each month. Tracks streaks of consecutive successful months.

### Baseline Calculation (set at goal creation)
```
// 3-month rolling average of category spend before goal creation
baseline_amount = avg(
    monthly_category_spend
    WHERE date >= goal_creation_date - 90 days
    AND date < goal_creation_date
    AND category_id = goal.category_id
)

// Target = baseline reduced by the chosen percentage
target_amount = baseline_amount × (1 - reduction_pct / 100)

// Example: $580/mo avg × (1 - 0.20) = $464/mo target
```

The baseline is stored at goal creation and never recalculated (it's a historical snapshot). This prevents gaming: if the user spends heavily to inflate the baseline, the goal remembers the original amount.

### Monthly Period Tracking
Each calendar month is a new period. The goal auto-renews at 00:01 on the 1st of each month.

```
// Current month progress
actual_this_month = sum(transactions WHERE category_id = goal.category_id
                                      AND date >= first_day_of_current_month
                                      AND date <= today)

// Expected spend at this point in the month (same period-marker logic as Dashboard Burn Rate)
expected_at_today = (day_of_month / days_in_month) × target_amount

// Month-end projection
projected_month_end = (actual_this_month / day_of_month) × days_in_month
```

### Status Logic
```
IF actual_this_month > target_amount:
    status = 'OVER'    // already failed this month

ELSE IF actual_this_month > (expected_at_today × 1.10):
    status = 'AT_RISK' // spending faster than safe pace

ELSE:
    status = 'ON_TRACK'
```

### Month Close Logic (runs at 00:01 on 1st of each month)
```
FOR each active Spending Habit goal:
    1. Read actual_spend for the closed month
    2. was_successful = (actual_spend <= target_amount)
    3. Insert goal_period_records row for the closed month
    4. IF was_successful:
           current_streak += 1
           IF current_streak > longest_streak: longest_streak = current_streak
           total_periods_hit += 1
           → trigger celebration notification
       ELSE:
           current_streak = 0  // streak broken
    5. total_periods_tracked += 1
    6. IF auto_renew = true: goal remains active for new month
       ELSE: goal status = 'completed'
```

### Streak Display Rules
```
IF current_streak == 0:   show nothing (no streak indicator)
IF current_streak == 1:   "1 month ✓"  (muted green)
IF current_streak 2-3:    "X months 🔥" (amber)
IF current_streak >= 4:   "X months 🔥" (brand colour — a real streak)
```

### AI-Generated Spending Habit Goals
Trigger conditions (background job):
```
IF avg(monthly_category_spend, last_3_months) > avg(monthly_category_spend, prev_3_months) × 1.15
    AND category is discretionary (not housing, utilities, insurance):
  → Generate suggestion for that category

IF category_spend > (budget.amount × 1.20) for 2+ consecutive months:
  → Generate "You've been over budget on X for 2 months — want to make a Spending Habit goal?"

IF user asks about high spend in AI Chat:
  → Generate goal preview in chat with calculated baseline and suggested reduction %
```

### Celebration Flow
When a month closes successfully:
```
1. Push notification (if enabled): "🎉 You hit your [category] goal this month! [streak] months in a row."
2. On next app open: goal card plays celebration animation (confetti burst)
3. If streak milestone hit (3, 6, 12 months): show full-screen celebration moment
```

### Edge Cases
| Scenario | Handling |
|---|---|
| < 3 months data for baseline | Use available data and label "based on X weeks of data" |
| CSV import provides 3+ months | Use imported data for baseline immediately |
| Category merged/renamed | Goal remains linked to category_id — survives renaming |
| User changes category budget mid-month | Goal target is independent of budget — they are separate |
| Zero spend month | Count as successful (actual $0 <= target) |

### Data Dependencies
- `goals` (base goal record)
- `goal_spending_habit` (baseline, target, reduction_pct, streak)
- `goal_period_records` (monthly history)
- `transactions` (current month category spend)
- `categories` (category name, icon)

---

## 3. Debt Paydown Goal

### What It Is
Links to a credit card or loan account and tracks debt elimination progress. Measures reduction in outstanding balance from the starting balance toward a target (usually $0).

### Formulas
```
// Amount paid off
amount_cleared = starting_balance - current_balance

// Progress
progress_pct = (amount_cleared / (starting_balance - target_balance)) × 100

// Projected payoff date at current monthly payment rate
avg_monthly_payment = avg(monthly_credit_card_payments, last_3_months)
months_to_payoff = current_balance / avg_monthly_payment
projected_payoff = today + months_to_payoff

// Months of interest saved (if paid by target date vs projected)
// displayed as motivational metric
interest_saved = current_balance × (interest_rate / 12) × months_difference
```

### Status Logic
```
IF projected_payoff < target_date - 30 days: status = 'AHEAD'
IF projected_payoff <= target_date:          status = 'ON_TRACK'
IF projected_payoff > target_date:           status = 'BEHIND'
IF no target_date set:                       status = 'IN_PROGRESS'
```

### Balance Tracking
- Current balance read from `accounts.balance` for the linked account (credit card)
- Updated on every Basiq account balance refresh
- Note: credit card balance in Basiq is typically a positive number representing money owed — must be confirmed during integration

### AI-Generated Debt Paydown Goals
Trigger conditions:
```
IF credit_card_account.balance > 0 AND user has no Debt Paydown goal for that account:
    AND credit_card_utilization > 30%:
  → Generate suggestion: "Your [card] balance is $X. Want to set a paydown goal?"
```

### Edge Cases
| Scenario | Handling |
|---|---|
| Balance increases (more spending) | Recalculate projected payoff — push notification if payoff date slips significantly |
| Balance hits $0 | Trigger goal completion, celebration |
| User has multiple credit cards | One goal per account — separate goals |
| Interest charges visible as transactions | Do not count interest as "spending" in category budgets — tag as `is_interest = true` |

### Data Dependencies
- `goals` (base goal record)
- `goal_debt_paydown` (starting_balance, target_balance, monthly_payment)
- `accounts` (current balance of linked credit card/loan)
- `transactions` (payment history for avg_monthly_payment calculation)

---

## 4. AI Goal Suggestions & the 30-Transaction Milestone

### The 30-Transaction Milestone
Gamified onboarding mechanic. Totem AI's categorisation engine activates after 30 reviewed transactions, which provides enough signal to generate meaningful suggestions.

```
reviewed_count = count(transactions WHERE reviewed = true)
milestone_pct = min(reviewed_count / 30, 1.0) × 100

// Display in Goals tab until milestone hit:
"Review X more transactions to activate Totem AI"

// On reaching 30:
1. Trigger full AI suggestion generation (run all suggestion algorithms)
2. Mark user as AI-active in user_settings
3. Show celebration: "Totem AI is now active" moment
4. Replace milestone card with first AI suggestion card
```

### AI Suggestion Card Display Rules
```
// Show when:
1. user.ai_active = true
2. pending AI suggestions exist (ai_goal_suggestions WHERE status = 'pending')
3. User has not seen this suggestion in last 7 days

// Show maximum 1 suggestion card at a time
// After user acts (creates goal or dismisses), show next pending suggestion (if any)

// Suggestion card fields:
- sparkle icon + "Totem AI" label
- insight text (why this goal is suggested)
- goal preview (type, target, calculation basis)
- [Review Suggestion] → opens goal creation sheet, pre-filled with AI data
- [Not Now] → sets suggestion.status = 'snoozed', resurface after 7 days
```

### Suggestion Generation Schedule
- On reaching 30 reviewed transactions (first time)
- Daily at 06:00 (check if new suggestions warranted)
- After any significant spending event (category spend >30% over average)
- After user closes a goal (check what to suggest next)

---

## 5. Budgets Sub-Tab

### What It Is
The management interface for category budget limits. These are the same budgets that feed the Dashboard Burn Rate widget. Setting a budget here immediately updates the Dashboard.

### Display Logic
```
// Summary header
total_budgeted = sum(budgets.amount WHERE is_active = true)
total_spent_mtd = sum(transactions WHERE is_income = false
                                    AND is_transfer = false
                                    AND date >= first_day_of_current_month)

// Per-category budget row
FOR each active budget:
    monthly_spend_mtd = sum(transactions WHERE category_id = budget.category_id
                                          AND date >= first_day_of_current_month)
    progress_pct = (monthly_spend_mtd / budget.amount) × 100
    expected_pct = (day_of_month / days_in_month) × 100
    // Same status logic as Dashboard Burn Rate (On Track / At Risk / Over)
```

### Period Context Banner
```
// Shown at the top of the budgets list
"Day X of February · X% of month elapsed"
// Helps users interpret burn rate bars without needing the period marker explanation
```

### Budget-to-Dashboard Sync
The Budgets sub-tab and Dashboard Burn Rate widget read from the same `budgets` table. Changes are immediately reflected on the Dashboard — no sync delay. When a user edits a budget limit in the Budgets tab:
1. Update `budgets.amount` in database
2. Invalidate the `budget_burn_rates` Redis cache key for this user
3. Next Dashboard open (or websocket push) shows updated rates

### Adding a Budget
```
Flow:
1. User taps "+ Add Budget"
2. Category selector bottom sheet (shows categories without an active budget)
3. User sets monthly amount
4. Optional: link to a Spending Habit goal (if one exists for this category)
   - If linked: budget limit = goal target_amount (synced)
5. Save → appears in Budgets list and Dashboard immediately
```

### Inline Editing
Budget amounts are directly editable in the list without navigating to a separate screen:
- Tap the budget amount → it becomes an editable field in-place
- Press done / tap away → save and update
- This reduces friction for small budget tweaks

### Budget + Spending Habit Goal Relationship
When a user has both a Spending Habit goal AND a budget for the same category:
```
// They can be linked or independent:
IF budget.linked_goal_id IS NOT NULL:
    budget.amount = goal_spending_habit.target_amount  // stay in sync
    // When goal renews or target changes, budget updates automatically

ELSE:
    budget and goal are independent — user manages both separately
```

### Data Dependencies
- `budgets` (budget amounts, category links)
- `transactions` (month-to-date spend per category)
- `categories` (names, icons)
- `goal_spending_habit` (for goal-budget sync, optional)

---

## 6. Proactive Notifications from Goals

### Save Up Goal Notifications
```
// Monthly contribution reminder (if no auto-transfer)
Trigger: Day of month when monthly_contribution is due AND no transfer detected yet
Message: "Don't forget your $X contribution to [Goal Name] this month"
Timing: 09:00 on the scheduled contribution day

// Goal completion imminent
Trigger: progress_pct >= 90%
Message: "You're 90% of the way to [Goal Name]! $X more to go."

// Behind schedule alert
Trigger: projected_completion_date drifts > 2 months past target_date
Message: "[Goal Name] is falling behind. You'd need $X/mo to still hit [date]."
```

### Spending Habit Goal Notifications
```
// At-risk alert (same logic as Dashboard AI Insight, but goal-specific)
Trigger: status = 'AT_RISK' AND days_remaining_in_month >= 5
Message: "Heads up — your [category] spending is running ahead. $X left this month."
Timing: 10:00 on the day the AT_RISK status is first detected

// Proactive high-risk day warning (Totem's differentiator)
Trigger: pattern detection identifies high-spend days for this category
         (e.g., the user consistently orders takeout on Thursdays)
Message: "Tomorrow's a high [category] day for you. Plan ahead to stay on track."
Timing: 18:00 the evening BEFORE the identified high-risk day

// Month-end stretch goal notification
Trigger: 5 days before month end AND on_track AND meaningful amount still to save
Message: "5 days to go — you're on track for your [category] goal! 🔥"

// Streak milestone notifications
Trigger: streak reaches 1 (first success), 3, 6, 12
Message: "3 months in a row under your [category] budget! You're building a habit. 🔥"
```

### Debt Paydown Goal Notifications
```
// Projected payoff date improvement
Trigger: projected_payoff_date improves by > 2 weeks (extra payment detected)
Message: "Great payment! You're now on track to pay off [card] by [new date]."

// Payoff imminent
Trigger: current_balance < (monthly_payment × 1.5)  — within 1.5 months
Message: "You're almost debt-free on [card]! $X left to go."
```

### Notification Quiet Hours
Respect `user_settings.notification_quiet_hours` (default 22:00–07:00).
The "proactive high-risk day" notification is always sent at 18:00 (outside quiet hours by default). If a user has a custom quiet window that includes 18:00, delay to 09:00 the same day.

---

## Appendix: Calculation Run Schedule

| Feature | Trigger |
|---|---|
| Save Up progress | On any transfer to linked account (Basiq webhook) + daily balance refresh |
| Spending Habit current month | On every new categorised transaction |
| Spending Habit month close | 00:01 on 1st of each month |
| Debt Paydown progress | On Basiq balance refresh (typically daily) |
| AI suggestion generation | Daily 06:00 + on 30-transaction milestone hit + on goal close |
| Budget burn rates | On every new transaction (same as Dashboard) |
| Streak calculation | On Spending Habit month close |
| High-risk day pattern detection | Weekly (Sunday 02:00) — detect recurring spend patterns for next 7 days |
