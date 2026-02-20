# Totem — Insights Feature List
## Math, Algorithms & Data Dependencies

> **Scope:** Insights screen only (Screen 5). 4-tab layout: Spend | Trends | Income | Forecast. All currency AUD/NZD. All amounts in cents unless noted.

---

## 1. Spend Tab

### 1.1 Period Selector

Three period chips: **This Month** / **3 Months** / **12 Months**. All metrics on the Spend tab recalculate based on the selected period.

```
Period date ranges:
  This Month:  first_day_of_current_month → today (partial month)
  3 Months:    today - 90 days → today (rolling 90 days)
  12 Months:   today - 365 days → today (rolling 365 days)
```

Default: **This Month** on tab open.

### 1.2 KPI Row

Three headline figures displayed as a horizontal row.

```
Total Spent:
  sum(transactions.amount WHERE is_income = false
                            AND is_transfer = false
                            AND date BETWEEN period_start AND today)

Daily Average:
  total_spent / days_elapsed_in_period
  // days_elapsed = (today - period_start) + 1

Saved %:
  ((total_income_in_period - total_spent_in_period) / total_income_in_period) × 100
  // total_income = sum(transactions WHERE is_income = true AND date IN period)
  // If income < spend: display as 0% (not negative)
```

**Alex's mock values (This Month, Feb 1–19):**
```
Total Spent:  $2,781
Daily avg:    $146   ($2,781 / 19 days)
Saved:        18%    (($3,240 income - $2,781 spend) / $3,240 × 100... approx; only one pay received)
```

### 1.3 Category Breakdown

Horizontal bars showing each category's spend as a percentage of monthly income (not as a percentage of total spend). This framing ties spending back to earnings rather than just showing composition.

```
For each category with spend > $0 in period:
  category_spend = sum(transactions.amount WHERE category_id = cat.id AND date IN period)
  pct_of_income  = (category_spend / reference_income) × 100

reference_income:
  This Month:  sum(income transactions this calendar month)
               → if partial month: use expected_monthly_income from users.expected_net_income
  3 Months:    avg_monthly_income over the period
  12 Months:   avg_monthly_income over the period

Display:
  Sort: by category_spend DESC (highest spend first)
  Show all categories with spend (no cap — scrollable list)
  Bar width: scaled to pct_of_income (bar at 100% = all income on one category)
  Max bar: capped at 100% width even if spend exceeds income

Colour coding:
  Default: var(--brand) bar
  Category over budget: var(--red) bar
  Any category > 30% of income: var(--amber) bar
  // Alex: Dining Out flagged red (over 5% threshold — design decision for this category)
```

### 1.4 Merchant Analysis (Top 5)

Shows the top 5 merchants by total spend in the selected period.

```
SELECT merchant_name, SUM(amount) AS total
FROM transactions
WHERE user_id = $1
  AND is_income = false
  AND is_transfer = false
  AND date BETWEEN period_start AND today
GROUP BY merchant_name_normalised
ORDER BY total DESC
LIMIT 5

Bar fill: (merchant_total / top_merchant_total) × 100
  → Top merchant always at 100% width; others are proportional
  → This is a proportional bar, NOT pct of income
```

**Alex's mock top merchants (Feb):**
```
1. Airbnb rent payment — $1,600 (housing)
2. Various Dining Out  — $341
3. Woolworths          — $178
4. Opal/Transport      — $122
5. Netflix + Spotify   — $27 (subscriptions)
```

### 1.5 AI Insight Card (Spend Tab)

Same card component as Dashboard AI Insight (see Dashboard/feature-list.md Section 4). On the Insights screen, the card is always shown (not hidden when there's nothing urgent). Content is the same highest-priority insight from `ai_insights` table.

```
IF no current insight: show a positive/neutral card:
  "Your spending looks healthy this month. Keep it up."
  card_variant = 'info'
```

### Data Dependencies
- `transactions` (spend by category, by merchant, income)
- `categories` (names, icons, colours)
- `budgets` (for over-budget flag on bars)
- `ai_insights` (top current insight)
- `pay_periods` or `users.expected_net_income` (reference income)

---

## 2. Trends Tab

The Trends tab provides longer-horizon analysis — where the user's money has been going over time, and how their financial health metrics are evolving. Unlike the Spend tab (which answers "this period"), the Trends tab answers "over time".

### 2.1 Lifestyle Creep (Activated State)

Full implementation of the Lifestyle Creep metric. Formula and logic are defined in `Dashboard/feature-list.md Section 10`. The Trends tab presents the full breakdown, whereas the Dashboard shows only the summary status.

**Activated state display:**
```
Primary metric card:
  creep_delta_pct: "+6%" (large, amber)
  Status label: "Watch This"
  Sub: "Your spending is growing 6% faster than your income"

Breakdown rows (2):
  "Spending growth (3mo vs 12mo)":   +11% → bar or metric
  "Income growth (3mo vs 12mo)":      +5% → bar or metric
  "Delta":                            +6% → highlighted

Main driver:
  Which category accounts for most of the spend growth?
  driver_category = category with highest (avg_spend_3mo - avg_spend_12mo) / avg_spend_12mo
  // Alex: Dining Out — up 34% vs 12mo average

States:
  < 3 months data:     "Building baseline (X of 3 months complete)"
  3–11 months data:    Show metric with "Based on X months" caveat
  12+ months data:     Full comparison with no caveat
```

### 2.2 Safety Net (Detail)

Same formula as Dashboard/feature-list.md Section 8. Trends tab shows the full breakdown, not just the headline months figure.

```
Hero metric:
  safety_net_months: 2.6 (amber — below 3-month target)

Breakdown:
  Liquid savings:          $12,450 (ANZ NetSaver)
  3-month avg expenses:    $4,788/month  ($14,363 / 3)
  Target (3 months):       $14,364
  Shortfall:               $2,100 (to reach 3-month target)

Progress bar:
  Fill: (safety_net_months / target_months) × 100
  // 2.6 / 3.0 = 87% filled
  Fill colour: amber (below target)
  Target marker: thin line at 100% (the 3-month target)
```

### 2.3 Monthly Spend Trend (5-Bar Chart)

A bar chart showing total spend per calendar month for the last 5 complete months (plus current partial month).

```
Data:
  FOR each month in [Oct, Nov, Dec, Jan, Feb]:
    monthly_spend = sum(transactions WHERE is_income = false
                                      AND is_transfer = false
                                      AND date_trunc('month', date) = month)

Bar height: proportional to max monthly spend
  bar_height_pct = (monthly_spend / max_monthly_spend) × 100

Colour rules:
  Current month (partial): bar shown with diagonal stripe pattern or dashed border → "partial"
  Month with highest spend: var(--red) bar   // Dec spike in holiday season
  All other months: var(--brand) bar

Labels:
  Bottom: month abbreviation (Oct, Nov, Dec, Jan, Feb)
  Top of bar: dollar amount ($X,XXX)
  Current month bar: "Feb*" with asterisk and "Partial" note below chart
```

**Alex's mock data:**
```
Oct: $3,890 (brand)
Nov: $4,120 (brand)
Dec: $5,640 (red — holiday spike)
Jan: $4,010 (brand)
Feb: $2,781 (partial, dashed)
```

### 2.4 Savings Velocity (Mini Chart)

5-month savings rate trend. Same formula as Dashboard/feature-list.md Section 9, visualised as bars.

```
Data:
  FOR each completed pay period in last 5 months:
    savings_rate_pct = (income - total_spend) / income × 100

Display:
  Small bar chart (shorter than spend trend — secondary metric)
  Bar colour: var(--brand)
  Current period bar: partial, lighter shade

Alex's mock:
  Oct: 22%
  Nov: 20%
  Dec: 10%  (holiday season)
  Jan: 27%  (peak — Jan resolutions)
  Feb: 18%  (current, partial)
```

### Data Dependencies
- `lifestyle_creep_snapshots` (historical creep data)
- `transactions` (3-month and 12-month averages for live calculation)
- `accounts` (safety net liquid savings)
- `recurring_bills` (committed next-month expenses for safety net)
- `pay_periods` (savings velocity per period)

---

## 3. Income Tab

### 3.1 KPI Row

```
Monthly (net):
  current month income = sum(transactions WHERE is_income = true
                                           AND date IN current calendar month)
  IF month not complete:
    show actual received + expected remaining = expected_monthly_income - received
    label with "(est.)" if showing expected

YTD (Year to Date):
  sum(transactions WHERE is_income = true
                     AND date >= first_day_of_current_year
                     AND date <= today)

Growth:
  avg_income_last_3mo = sum(income last 3 months) / 3
  avg_income_prior_3mo = sum(income months 4–6 ago) / 3
  growth_pct = (avg_income_last_3mo - avg_income_prior_3mo) / avg_income_prior_3mo × 100

  Display: "+5%" (green if positive, red if negative, neutral if 0)
```

**Alex's mock values:**
```
Monthly: $6,400  (expected $6,480 — one of two Feb pays landed)
YTD: $9,640      ($3,200 Jan pay 1 + $3,200 Jan pay 2 + $3,240 Feb pay 1)
Growth: +5%
```

### 3.2 Dual Bar Chart (Income vs Spend)

A side-by-side bar chart showing total income (brand bar) vs total spend (red bar) for each of the last 5 months.

```
For each month:
  income_bar_height = (monthly_income / max_income_across_5mo) × 100
  spend_bar_height  = (monthly_spend  / max_income_across_5mo) × 100
  // Both bars scaled against the same max (income) so spend bars are comparable to income

Bar width: each month gets equal width, with income + spend bars side-by-side (small gap)

Special cases:
  Current month (partial):
    Income bar: dashed or lighter fill, shows only received income
    Spend bar: solid — actual spend so far (partial month)
    Label: "Feb (partial)" below

Surplus/deficit shading:
  If spend bar < income bar: gap between tops is implicit surplus (no explicit shading needed)
  If spend bar > income bar: spend bar overflows above income bar in red

Labels:
  Income bar: no label (use legend)
  Spend bar: no label
  Legend below chart: "■ Income  ■ Spend" (brand + red)
```

**Alex's mock data:**
```
Oct: Income $6,400, Spend $3,890
Nov: Income $6,400, Spend $4,120
Dec: Income $6,400, Spend $5,640
Jan: Income $6,400, Spend $4,010
Feb: Income $3,240 (partial, dashed), Spend $2,781
```

### 3.3 Pay History

Last 4 income transactions from the primary employer, shown as a simple list.

```
SELECT t.date, t.amount, t.merchant_name
FROM transactions t
WHERE t.user_id = $1
  AND t.is_income = true
  AND t.merchant_name_normalised = users.income_source_name
ORDER BY t.date DESC
LIMIT 4
```

Display per row:
```
Left: pay source name + date  (e.g. "Acme Corp · 8 Feb")
Right: amount (e.g. "$3,240")
Sub: pay period label (e.g. "Fortnightly pay")
```

### 3.4 Savings Breakdown

Shows how the user's savings are allocated across goals and accounts in the current month.

```
For each savings_goal WHERE auto_transfer_enabled = true:
  Show: goal name, this month's transfer amount vs scheduled amount
  e.g. "Japan Trip: $280 of $300 this month"

For each savings account transfer (non-goal):
  Show: account name, amount transferred this month
  e.g. "ANZ NetSaver: $500 transferred"
```

### Data Dependencies
- `transactions` (income records, pay history)
- `pay_periods` (expected vs actual income)
- `users` (expected_net_income, income_source_name)
- `savings_goals` (auto-transfer schedules)
- `accounts` (savings accounts)

---

## 4. Forecast Tab

### 4.1 Month-End Projection (Hero)

Same core formula as Dashboard Month End Projection (Dashboard/feature-list.md Section 2), presented here as the hero metric of the Forecast tab.

```
projected_surplus =
    expected_remaining_income_this_month
  + income_received_this_month
  - confirmed_bills_remaining_this_month
  - projected_discretionary_spend_remaining_this_month

projected_discretionary_remaining =
    (actual_discretionary_mtd / days_elapsed) × days_remaining_in_month

projected_total_spend = actual_spend_mtd + projected_discretionary_remaining + bills_remaining
projected_surplus = expected_total_income - projected_total_spend
```

**Alex's mock values (Feb 19):**
```
Expected total Feb income:   $6,480  (one $3,240 pay received, one due Feb 22)
Actual spend so far:         $2,781
Projected remaining spend:   $1,458  ($146/day × 10 remaining days)
Known remaining bills:       Rent $1,600 Feb 21 → already captured in total
                             Netflix $19.99 Feb 20
                             Spotify $12.99 Feb 22

Projected total spend:       ~$4,239
Projected surplus:           $6,480 - $4,239 = +$2,241
```

Display:
```
Hero card:
  Label: "Month-end projection"
  Amount: "+$2,241" (large, brand or green)
  Sub: "Income $6,480 · Projected spend $4,239"
  Status badge: "On Track" (green)
```

### 4.2 90-Day Cash Flow Projection

Three paired bars (income vs projected spend) for the next 3 calendar months. Projects forward using:

```
For each future month M (Mar, Apr, May):

  projected_income_M:
    IF user has regular pay cycle:
      count paydays in month M × expected_net_per_pay
    ELSE:
      use avg_monthly_income_3mo

  projected_spend_M:
    base_projection = avg_monthly_spend_3mo

    Adjustments (additive):
      + known_recurring_bills_in_M (from recurring_bills WHERE next_due_date IN month M)
      + goal_auto_transfers_in_M (from savings_goals WHERE next_transfer_date IN month M)

    Seasonal adjustment (if 12+ months history):
      prior_year_same_month_delta = spend_M_last_year - avg_spend_12mo_last_year
      projected_spend_M += prior_year_same_month_delta × 0.5  // dampen prior year anomaly by 50%

  projected_surplus_M = projected_income_M - projected_spend_M
```

**Alex's mock projections:**
```
Mar: Income ~$6,480, Spend ~$4,100, Surplus ~+$2,380
Apr: Income ~$6,480, Spend ~$4,200, Surplus ~+$2,280
May: Income ~$6,480, Spend ~$4,200, Surplus ~+$2,280
Avg projected surplus: ~$2,180/month
```

### 4.3 Goal ETAs

For each active `Save Up` goal, compute the projected date at which the goal will be fully funded, given the current monthly contribution rate.

```
For each active goal of type 'save_up':
  remaining = goal.target_amount - goal.current_amount
  monthly_contribution:
    IF goal.auto_transfer_enabled:
      use goal.auto_transfer_amount × (12 / period_multiplier) to get monthly equivalent
    ELSE:
      avg_monthly_contribution = avg(goal_contributions.amount) over last 3 months
      monthly_contribution = avg_monthly_contribution

  months_remaining = ceil(remaining / monthly_contribution)
  eta_date = today + months_remaining months

  completion_pct = (goal.current_amount / goal.target_amount) × 100
```

Display:
```
Goal name
Progress bar (current/target)
ETA date label: "May 2026" (brand colour)
Completion %: shown as badge  // "68%"
```

**Alex's mock goals:**
```
Japan Trip:     $2,840 / $4,200 = 68%  → ETA May 2026   (saving ~$280/month)
Safety Net 3mo: $12,450 / $14,550 = 86% → ETA Apr 2026  (needs ~$2,100 more)
New Car deposit: $3,100 / $10,000 = 31% → ETA Dec 2026  (saving ~$500/month)
```

### 4.4 Upcoming Known Expenses

Bills from `recurring_bills` and manually logged one-off expenses, shown in a list ordered by date.

```
SELECT
  rb.merchant_name,
  rb.next_due_date,
  rb.amount,
  rb.category_id,
  'recurring' AS source
FROM recurring_bills rb
WHERE rb.user_id = $1
  AND rb.status = 'confirmed'
  AND rb.next_due_date BETWEEN today AND today + 90 days

ORDER BY next_due_date ASC
LIMIT 10

// Future: also include manually-logged one-off expenses (not yet in MVP)
```

**Alex's mock upcoming (next 90 days):**
```
Feb 21:  Rent             $1,600
Mar 15:  Car registration   $680
Apr 1:   Japan flights deposit $1,200
```

### 4.5 AI Risk Flags

A list of 2–5 forward-looking AI-generated flags, scored and ordered by priority. These are distinct from the dashboard AI insight — they are forward-looking and specific to the forecast horizon.

```
Flag types:
  'spend_risk':       Category trending above budget on pace to overspend
  'known_expense':    Large known upcoming expense within 30 days
  'goal_opportunity': User is ahead of pace on a goal — could contribute more
  'milestone_close':  A goal or safety net milestone is within 1 month of being hit

Flag score = (urgency × 0.4) + (dollar_impact_score × 0.35) + (time_horizon_score × 0.25)
  time_horizon_score: < 7 days = 3, 7–30 days = 2, 31–90 days = 1

Colour coding:
  'spend_risk':       red flag   (var(--red))
  'known_expense':    amber flag (var(--amber))
  'goal_opportunity': green flag (var(--green))
  'milestone_close':  brand flag (var(--brand))
```

**Alex's mock flags (4 items):**
```
1. 🔴 Dining Out Risk — "On pace to overspend by ~$120 this month. Cut 2 meals out."
2. 🟡 Car Rego — "Car registration $680 due Mar 15. Allocating from everyday account."
3. 🟢 Japan Trip — "Ahead of pace by $160. You could hit goal 3 weeks early."
4. 🔵 Safety Net Milestone — "You're 86% to your 3-month safety net. ETA Apr 2026."
```

### Data Dependencies
- `transactions` (spend averages, income figures)
- `pay_periods` (pay dates, expected income)
- `recurring_bills` (known upcoming bills)
- `savings_goals` (auto-transfer amounts, current balances for goal ETAs)
- `goal_contributions` (historical contribution rates for ETA calculation)
- `accounts` (safety net current balance)
- `ai_insights` (risk flags — generated and scored separately, read here)
- `lifestyle_creep_snapshots` (seasonal spend adjustment for 90-day forecast)

### Calculation Run Schedule
- Month-end projection: on every new transaction + app open (same as Dashboard)
- 90-day cash flow: daily at 06:00 + on app open + after any recurring bill change
- Goal ETAs: on every goal contribution + daily
- AI risk flags: daily at 06:00 + on every new transaction

---

## Appendix: Calculation Run Schedule

| Feature | Trigger |
|---|---|
| Spend tab KPIs + category breakdown | On every new transaction + app open |
| Merchant analysis | On every new transaction + app open |
| Lifestyle Creep (Trends) | Monthly snapshot (1st at 02:00) — same as dashboard |
| Safety Net detail | On every new transaction + daily balance refresh |
| Monthly Spend Trend chart | On month rollover + app open |
| Savings Velocity chart | On pay period close |
| Income tab KPIs + dual bar | On every income transaction + app open |
| Pay history | On every income transaction |
| Month-end projection hero | On every new transaction + app open |
| 90-day cash flow | Daily at 06:00 + app open + recurring bill changes |
| Goal ETAs | On every goal contribution + daily |
| AI risk flags | Daily at 06:00 + every new transaction |
