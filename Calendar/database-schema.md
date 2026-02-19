# Totem — Calendar Screen Database Schema
## Calendar Data Architecture

> **Important:** The Calendar screen is primarily a **read layer** over existing tables (`transactions`, `recurring_bills`, `pay_periods`, `budgets`, `categories`). The only new table introduced is `calendar_day_cache` — a pre-computed daily summary table that makes calendar rendering fast without re-aggregating transactions on every open.
> All amounts in INTEGER cents. All timestamps TIMESTAMPTZ.

---

## New Table

### `calendar_day_cache`
Pre-computed daily spend/income totals per user. Populated by webhooks and scheduled jobs. Falls back to live query if cache is stale.

```sql
CREATE TABLE calendar_day_cache (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Date (one row per user per calendar day)
    calendar_date         DATE NOT NULL,

    -- Aggregated totals (in cents)
    spend_total_cents     INTEGER NOT NULL DEFAULT 0,
        -- Sum of all outgoing non-transfer transactions for this date
    income_total_cents    INTEGER NOT NULL DEFAULT 0,
        -- Sum of all income transactions for this date (wages, etc.)
    net_cents             INTEGER NOT NULL DEFAULT 0,
        -- income_total_cents - spend_total_cents

    -- Category breakdown (top 3 by spend, for dot rendering)
    top_category_1_id     UUID REFERENCES categories(id),
    top_category_1_cents  INTEGER,
    top_category_2_id     UUID REFERENCES categories(id),
    top_category_2_cents  INTEGER,
    top_category_3_id     UUID REFERENCES categories(id),
    top_category_3_cents  INTEGER,

    -- Safety tint (pre-computed, avoids budget join on every render)
    daily_budget_cents    INTEGER,
        -- Snapshot of user's daily budget at time of computation
        -- = sum(budgets.amount WHERE category.is_discretionary) / days_in_month
    tint_level            VARCHAR(10),
        -- 'green' | 'amber' | 'red' | 'payday' | NULL

    -- Transaction count
    transaction_count     SMALLINT NOT NULL DEFAULT 0,
    has_income            BOOLEAN NOT NULL DEFAULT FALSE,

    -- Cache management
    computed_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_stale              BOOLEAN NOT NULL DEFAULT FALSE,

    -- Constraint: one row per user per day
    UNIQUE(user_id, calendar_date)
);
```

---

## Reused Tables (no schema changes)

### `transactions` — primary data source
Key fields used by Calendar:

| Field | Used For |
|---|---|
| `occurred_at` | grouping by calendar_date |
| `amount_cents` | daily totals, dot sizing |
| `is_income` | income day detection, net calc |
| `is_transfer` | excluded from spend totals |
| `category_id` | category dots |
| `merchant_name_normalised` | day detail sheet display |
| `user_id` | filtering |

```sql
-- Calendar-specific query: fetch a single day's transactions
SELECT
    t.id,
    t.occurred_at::time AS txn_time,
    t.amount_cents,
    t.is_income,
    t.merchant_name_normalised,
    c.name AS category_name,
    c.emoji_icon,
    c.colour_hex
FROM transactions t
JOIN categories c ON c.id = t.category_id
WHERE t.user_id = $1
  AND t.occurred_at::date = $2
  AND t.is_transfer = false
  AND t.deleted_at IS NULL
ORDER BY t.is_income DESC, t.amount_cents DESC;
```

---

### `recurring_bills` — upcoming bill markers
Key fields used by Calendar:

| Field | Used For |
|---|---|
| `next_due_date` | which future cell gets a bill flag |
| `amount_cents` | amount shown in bill detail |
| `merchant_name_normalised` | bill name in detail sheet |
| `status` | only 'confirmed' bills shown |
| `confidence_level` | 'high' and 'medium' shown; 'low' hidden |

```sql
-- Fetch bills due within a date range (for a calendar month render)
SELECT
    rb.id,
    rb.next_due_date,
    rb.amount_cents,
    rb.merchant_name_normalised,
    rb.confidence_level,
    c.emoji_icon
FROM recurring_bills rb
LEFT JOIN categories c ON c.id = rb.category_id
WHERE rb.user_id = $1
  AND rb.next_due_date BETWEEN $2 AND $3  -- month start/end
  AND rb.status = 'confirmed'
  AND rb.confidence_level IN ('high', 'medium')
  AND rb.deleted_at IS NULL
ORDER BY rb.next_due_date;
```

---

### `pay_periods` — payday markers
Key fields used by Calendar:

| Field | Used For |
|---|---|
| `start_date` | payday marker date on calendar |
| `expected_income_cents` | "~$3,200" on future payday cells |
| `actual_income_cents` | confirmed income on past payday cells |
| `user_id` | filtering |

```sql
-- Fetch pay dates overlapping or adjacent to current month
SELECT
    start_date,
    expected_income_cents,
    actual_income_cents
FROM pay_periods
WHERE user_id = $1
  AND start_date BETWEEN $2 - INTERVAL '14 days' AND $3 + INTERVAL '14 days'
ORDER BY start_date;
```

---

### `budgets` — daily budget denominator for tint calculation
Key fields used by Calendar:

| Field | Used For |
|---|---|
| `amount` | numerator of daily_budget calc |
| `category_id → categories.is_discretionary` | filter: only discretionary budgets count |
| `is_active` | only active budgets |
| `period_type` | only 'monthly' budgets (daily = divide by days_in_month) |

```sql
-- Compute user's daily discretionary budget for a given month
SELECT SUM(b.amount) / $3 AS daily_budget_cents  -- $3 = days_in_month
FROM budgets b
JOIN categories c ON c.id = b.category_id
WHERE b.user_id = $1
  AND b.is_active = true
  AND b.period_type = 'monthly'
  AND c.is_discretionary = true;
```

---

## Indexes

```sql
-- calendar_day_cache: primary access pattern (month range per user)
CREATE INDEX idx_cal_cache_user_date  ON calendar_day_cache(user_id, calendar_date DESC);
CREATE INDEX idx_cal_cache_stale      ON calendar_day_cache(user_id, is_stale) WHERE is_stale = true;

-- transactions: calendar date lookup (hot path)
CREATE INDEX idx_txns_user_date       ON transactions(user_id, (occurred_at::date));
-- Note: if this index doesn't exist already in Dashboard schema, add via migration

-- recurring_bills: upcoming bills for a date range
CREATE INDEX idx_recur_bills_due      ON recurring_bills(user_id, next_due_date)
    WHERE status = 'confirmed' AND deleted_at IS NULL;

-- pay_periods: payday lookup
CREATE INDEX idx_pay_periods_user_date ON pay_periods(user_id, start_date DESC);
```

---

## Cache Invalidation Rules

The `calendar_day_cache` is updated by the following events:

| Event | Action |
|---|---|
| **Basiq webhook: new transaction** | Recompute the affected `calendar_date` row |
| **User re-categorises transaction** | Mark affected `calendar_date` row `is_stale = true`; recompute |
| **Budget amount changed** | Mark all future rows for current month `is_stale = true` (tint recalculation) |
| **App open** | Check for any stale rows in the current 60-day window; recompute if found |
| **Month navigation to past month** | Cache miss → compute on-demand and cache |

### Fallback
If `calendar_day_cache` is empty or stale for a day, the API falls back to a live aggregation query directly on `transactions`. Latency impact: 80–200ms vs <10ms from cache.

---

## Calendar API Endpoint Payload

```
GET /api/v1/calendar?month=2026-02&user_id=...
```

### Response
```json
{
  "month": "2026-02",
  "days_in_month": 28,
  "month_start_day_of_week": 0,
  "daily_budget_cents": 3325,
  "month_total_spend_cents": 115700,
  "prev_month_same_day_spend_cents": 107100,
  "vs_prev_pct": 8.0,
  "days": [
    {
      "date": "2026-02-01",
      "spend_total_cents": 4500,
      "income_total_cents": 0,
      "net_cents": -4500,
      "has_income": false,
      "is_payday": false,
      "tint_level": "green",
      "transaction_count": 2,
      "category_dots": [
        { "category_id": "...", "colour_hex": "#51CF66" },
        { "category_id": "...", "colour_hex": "#4ECDC4" }
      ]
    }
    ...
  ],
  "upcoming_bills": [
    { "date": "2026-02-20", "merchant": "Netflix", "amount_cents": 1899, "urgency": "urgent" },
    { "date": "2026-02-21", "merchant": "Rent", "amount_cents": 100000, "urgency": "urgent" },
    { "date": "2026-02-22", "merchant": "Spotify", "amount_cents": 1299, "urgency": "normal" }
  ],
  "payday_dates": ["2026-02-08", "2026-02-22"],
  "today": "2026-02-19"
}
```

---

## Feature → Table Dependency Map

| Feature | Primary Tables | Secondary |
|---|---|---|
| **Monthly grid render** | `calendar_day_cache` | `transactions` (fallback) |
| **Safety tinting** | `calendar_day_cache.tint_level` | `budgets`, `categories` |
| **Category dots** | `calendar_day_cache.top_category_*` | `categories` (colour) |
| **Payday markers** | `pay_periods` | `transactions` (income detection) |
| **Bill flags** | `recurring_bills` | `categories` (icon) |
| **Day detail sheet** | `transactions` | `categories`, `recurring_bills` |
| **Month summary banner** | `calendar_day_cache` (aggregated) | — |
| **vs Last Month %** | `calendar_day_cache` (two months) | — |

---

## Data Flow Notes

### Calendar Screen Load
```
1. GET /api/v1/calendar?month=current_month
2. Check calendar_day_cache for all days in month:
   a. Hit (not stale): return cached row
   b. Miss or stale: aggregate from transactions → insert/update cache → return
3. Render grid with tint levels, dots, payday markers, bill flags
4. Pre-fetch adjacent month cache in background (for instant nav)
```

### Day Tap → Detail Sheet
```
1. User taps cell for date X
2. GET /api/v1/calendar/day?date=2026-02-19&user_id=...
3. SELECT transactions WHERE date = X (live query, not cached — real-time accuracy needed)
4. SELECT recurring_bills WHERE next_due_date = X (for future days)
5. Return sorted transaction list + bill list
6. Render bottom sheet content
```

### Cache Recomputation (single day)
```
1. Trigger: new transaction webhook for user U on date D
2. SELECT sum(amount_cents) FROM transactions WHERE user_id = U AND date = D AND is_income = false
3. SELECT sum(amount_cents) FROM transactions WHERE user_id = U AND date = D AND is_income = true
4. SELECT top 3 category_ids by spend for that day
5. Compute tint_level against daily_budget_cents
6. UPSERT calendar_day_cache (user_id, calendar_date) with new values
7. Push WebSocket event to connected clients: { type: 'cal_day_update', date: D }
```
