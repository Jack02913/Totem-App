# Totem — Calendar Feature List
## Math, Algorithms & Data Dependencies

> **Scope:** Calendar screen only. All values are per-user. All currency in AUD (NZ flow identical with NZD).

---

## 1. Monthly Calendar Grid

### What It Is
A 7-column (Sun–Sat) month grid where each cell represents a calendar day. Past days show actual spend; future days show upcoming bills and expected income. The current day is highlighted. Users tap any day cell to open a detail bottom sheet.

### Grid Logic
```
Days per month: standard calendar month (28/29/30/31 days)
Columns: 7 (Sun=0 through Sat=6)
Leading blank cells: (day_of_week(month_start)) empty cells before day 1
Rows: ceil((leading_blanks + days_in_month) / 7)
     → Feb 2026: 0 blanks + 28 days = 4 rows (perfect grid, no blanks)
     → Most months: 4 or 5 rows; rare 6-row month possible
```

### Day Cell States
| State | Trigger | Visual |
|---|---|---|
| **Past** | date < today | actual spend shown, safety tint applied |
| **Today** | date = today | brand-coloured date number ring, partial spend shown |
| **Future** | date > today | dimmed (opacity 0.7), bill flags shown if applicable |
| **Payday** | date matches next/prev pay date | brand-tinted background, payday label inside cell |
| **No Activity** | past day with $0 spend | neutral background, no amount shown |

### Data Dependencies
- `transactions` (date, amount, category_id)
- `pay_periods` (next_payday, period boundaries)
- `recurring_bills` (next_due_date, amount)
- `calendar_day_cache` (pre-computed daily totals for rendering speed)

---

## 2. Daily Spend Safety Tinting

### What It Is
Totem's primary calendar differentiator — subtle background colour on each past day cell indicating whether that day's spend was under, on-pace, or over budget relative to the daily discretionary budget. Makes spending patterns instantly scannable at a glance without reading any numbers.

### Daily Budget Formula
```
daily_discretionary_budget = sum(budgets.amount WHERE category.is_discretionary = true) / days_in_month
```

### Tint Levels
| Tint | Condition | Background |
|---|---|---|
| **Green** (Under) | actual_spend ≤ 80% of daily_budget | rgba(81,207,102,0.09) |
| **Amber** (On Pace) | 80% < actual_spend ≤ 120% of daily_budget | rgba(247,200,75,0.09) |
| **Red** (Over) | actual_spend > 120% of daily_budget | rgba(255,107,107,0.10) |
| **Brand** (Payday) | income received this day | rgba(139,126,255,0.13) |
| **None** | $0 spend, or future day | transparent |

### Edge Cases
| Scenario | Handling |
|---|---|
| No budgets set | No tinting applied; cells show spend only |
| Payday + high spend same day | Brand tint takes priority (income day dominates) |
| Lump-sum bill day (e.g. rent) | Excluded from daily tint calc if category is `housing` and flagged as `is_recurring` |
| Irregular income user | Tinting uses 30-day rolling average spend as denominator |

### Data Dependencies
- `budgets` (amount, category.is_discretionary)
- `calendar_day_cache` (daily_total_cents)

---

## 3. Category Dot Composition

### What It Is
Up to three coloured dots at the bottom of each day cell, representing the top spending categories for that day (by spend amount). Gives a visual fingerprint of each day's spending type — a day with red+amber+teal dots is an "eating out + entertainment + transport" day at a glance.

### Algorithm
```
FOR each past day:
  categories = SELECT category_id, SUM(amount) as day_total
               FROM transactions
               WHERE date = day AND is_income = false
               GROUP BY category_id
               ORDER BY day_total DESC
               LIMIT 3
  dots = categories.map(c => c.colour)
```

### Category Colour Mapping
| Category | Dot Colour | CSS Variable |
|---|---|---|
| Eating Out / Dining | Red | `var(--red)` |
| Groceries / Supermarket | Green | `var(--green)` |
| Transport | Teal | `var(--teal)` |
| Entertainment / Leisure | Amber | `var(--amber)` |
| Subscriptions / Software | Brand purple | `var(--brand)` |
| Housing / Rent | White (muted) | `rgba(255,255,255,0.3)` |
| Health | Teal (lighter shade) | custom |
| Shopping / Retail | Amber | `var(--amber)` |

### Overflow
- If a day has >3 categories: show 3 dots only (top 3 by spend amount). The detail sheet on tap shows all.

### Data Dependencies
- `transactions` (date, category_id, amount)
- `categories` (colour_hex)

---

## 4. Payday Marker

### What It Is
Visual marker on the calendar grid showing when pay is deposited. Critical for the "pay-period framing" mental model — users can immediately see how far they are from the next payday and how much the last pay period covered.

### Logic
```
NEXT payday: next date in pay_periods WHERE period_start > TODAY
LAST payday: max date in pay_periods WHERE period_start <= TODAY

Both dates receive a payday cell treatment:
  - Brand-tinted background (rgba(139,126,255,0.13))
  - "Payday" micro-label inside the cell
  - Income amount shown if past (actual), "~$X" if future (expected)
```

### Pay Cycle Handling
- **Detected cycle** (fortnightly/weekly/monthly): auto-populates future payday dates for the next 3 months
- **Irregular income**: no future payday marker; past income deposits shown with brand tint

### Data Dependencies
- `pay_periods` (start_date, expected_income, actual_income)
- `transactions` (to detect income deposits — is_income = true, amount > threshold)

---

## 5. Bill & Upcoming Payment Indicators

### What It Is
Small coloured dot in the top-right corner of future day cells when a confirmed recurring bill is due on that date. Colour signals urgency. Tapping the cell opens the detail sheet listing the upcoming bills.

### Urgency Logic
```
bill_due_date = recurring_bills.next_due_date
days_until = bill_due_date - TODAY

IF days_until <= 3  → red flag dot   (urgent)
IF days_until 4-7   → amber flag dot (coming up)
IF days_until > 7   → amber flag dot (normal)
```

### Bill Data Source
Only `recurring_bills.status = 'confirmed'` bills appear. Medium/low confidence bills are hidden (same logic as Dashboard Upcoming Bills widget).

### Data Dependencies
- `recurring_bills` (next_due_date, status, amount, merchant_name_normalised)

---

## 6. Day Detail Bottom Sheet

### What It Is
A bottom sheet (slides up from bottom) triggered by tapping any day cell. Shows all transactions for that day, and for future days shows upcoming bills. This is the primary drill-down interaction on the Calendar screen.

### Past Day Content
```
Header: "Thursday, 19 Feb 2026"
Summary row: "Total spent · −$54"

FOR each transaction ON that day (ordered by time DESC):
  - Merchant icon + name
  - Category + time
  - Amount (negative for spend, positive for income)

IF is_income_day:
  - Income row at top: "ANZ Payroll · +$3,200"

Footer: "See all in Accounts →" link
```

### Future Day Content
```
Header: "Friday, 21 Feb 2026"
Badge: "Future"

Section: "Upcoming Bills"
FOR each recurring_bill WHERE next_due_date = that day AND status = 'confirmed':
  - Icon + merchant name + Auto-debit label + amount

IF is_payday:
  - "Expected income: ~$3,200" row at top

Footer: "Nothing else scheduled on this day" if no transactions/bills
```

### Transaction Display Order
1. Income transactions (top, brand-coloured)
2. Largest expense first (descending by amount)

### Data Dependencies
- `transactions` (merchant_name, amount, category_id, occurred_at)
- `recurring_bills` (next_due_date, amount, merchant_name_normalised)
- `categories` (name, emoji_icon, colour_hex)

---

## 7. Month Navigation

### What It Is
Left/right chevron buttons in the calendar header allow the user to navigate between months. Up to 6 months of transaction history is available (limited by Basiq data retention). The current month is the default view on screen load.

### Navigation Logic
```
Current month (default): TODAY's month/year
Previous month: TODAY's month - 1 (up to 6 months back)
Future months: TODAY's month only (no forward navigation; future months have no data)

Exception: if next_payday is in the next calendar month, the "upcoming bills"
for those days at month boundary are visible in the current month view's future cells.
```

### Month Summary Update
When the user navigates to a different month, the three summary cells update:
- **Spent this month**: total spend for that month (or partial if current)
- **vs Last Month**: % change vs the immediately preceding month
- **Days Remaining**: only shown for current month; past months show "31 days" (total)

### Data Dependencies
- `calendar_day_cache` (month-level aggregates)
- `transactions` (for accurate monthly totals)

---

## 8. Month Summary Banner

### What It Is
A three-cell summary strip below the month navigation header showing key month-level metrics at a glance: total spend, spend change vs previous month, and days remaining.

### Cells
| Cell | Label | Value | Logic |
|---|---|---|---|
| **Spent** | "Spent" | `$1,157` | Sum of all spending transactions this month to date |
| **vs Last Month** | "vs Last Mo." | `+8%` | (this_month_spend_to_same_day - last_month_same_day_spend) / last_month_same_day_spend × 100. Red if +%, green if −%. |
| **Days Remaining** | "Days Left" | `9` | days_in_month − today's date. For past months: shows total days. |

### "Same Day" Comparison Logic
To avoid misleading comparisons (e.g., comparing 19 days of Feb vs full 31 days of Jan), the vs-last-month % compares spend through day N of the current month vs spend through day N of the previous month.

```
vs_last_month_pct =
  (spend_this_month_through_today - spend_last_month_through_same_day)
  / spend_last_month_through_same_day × 100
```

### Data Dependencies
- `calendar_day_cache` (daily totals for both months)

---

## 9. Net Day Indicator (Income Days)

### What It Is
On days where income was received (payday), the day cell amount shows the net figure (income minus spend on that day) in brand purple, rather than a negative spend figure. This prevents confusion on paydays — seeing "+$3,166" instead of "−$34" on the day $3,200 was received.

### Formula
```
IF sum(transactions WHERE date = day AND is_income = true) > 0:
  net = income_total - spend_total
  display_amount = "+" + net (brand purple)
ELSE:
  display_amount = "−" + spend_total (white/red based on tint)
```

### Data Dependencies
- `transactions` (is_income, amount, date)
