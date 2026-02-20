# Totem — Database Schema
## Insights Screen Data Architecture

> **Cross-reference:** The Insights screen is primarily a **read-heavy analytics layer** on top of tables defined in `Dashboard/database-schema.md` and `Plan/database-schema.md`. This file documents only the **additional tables** that the Insights screen introduces for performance and caching.
>
> **Tables from Dashboard schema read by Insights:**
> - `transactions` — all spend, income, and transfer records
> - `categories` — category names, icons, colours
> - `budgets` — budget amounts per category (for over-budget flags on bars)
> - `accounts` — balances (for safety net liquid savings)
> - `recurring_bills` — confirmed bills (for upcoming known expenses + 90-day forecast)
> - `ai_insights` — highest-priority insight (shown on Spend tab)
> - `lifestyle_creep_snapshots` — historical Lifestyle Creep data (Trends tab)
> - `pay_periods` — income and period framing
> - `computed_metrics_cache` — pre-computed metric fallback
>
> **Tables from Plan schema read by Insights:**
> - `goals` + `goal_save_up` + `goal_contributions` — goal ETAs (Forecast tab)
>
> **Database:** PostgreSQL (primary). Redis for caching computed Insights metrics.
> **Conventions:** UUID PKs, TIMESTAMPTZ, INTEGER cents, soft deletes via `deleted_at`.

---

## Table of Contents
1. [Metric Snapshots](#metric-snapshots)
2. [Forecast Cache](#forecast-cache)
3. [Spend Trends Cache](#spend-trends-cache)
4. [Feature → Table Dependency Map](#feature--table-dependency-map)
5. [Indexes](#indexes)
6. [Data Flow Notes](#data-flow-notes)

---

## Metric Snapshots

### `metric_snapshots`
Period-by-period snapshots of all key Insights metrics. Populated nightly. Enables fast chart rendering (Trends tab bar charts, Income tab dual bar) without full transaction re-aggregation on each page load.

```sql
CREATE TABLE metric_snapshots (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Period this snapshot covers
    period_type           VARCHAR(20) NOT NULL,  -- 'monthly' | 'weekly' | 'pay_period'
    period_start          DATE NOT NULL,
    period_end            DATE NOT NULL,

    -- Spend metrics
    total_spend           INTEGER,       -- cents, all non-income, non-transfer transactions
    discretionary_spend   INTEGER,       -- cents, non-recurring spend only
    recurring_spend       INTEGER,       -- cents, confirmed recurring bills

    -- Income metrics
    total_income          INTEGER,       -- cents
    net_income            INTEGER,       -- cents (total_income - total_spend)

    -- Savings
    savings_rate_pct      NUMERIC(5,2),  -- (net_income / total_income) × 100

    -- Category breakdown (top 5 by spend, stored as JSONB for flexibility)
    category_breakdown    JSONB,
    -- e.g. [{"category_id": "uuid", "name": "Eating Out", "amount": 34100, "pct_of_income": 5.3}, ...]

    -- Merchant breakdown (top 5)
    merchant_breakdown    JSONB,
    -- e.g. [{"merchant_name": "Woolworths", "amount": 17800}, ...]

    -- Averages (rolling at the time of snapshot)
    avg_daily_spend       INTEGER,       -- cents (total_spend / days_in_period)

    -- Lifestyle creep inputs (stored here for chart without recalculating)
    avg_spend_3mo_at_period_end  INTEGER,  -- cents
    avg_income_3mo_at_period_end INTEGER,

    -- Data completeness
    is_partial            BOOLEAN DEFAULT FALSE,  -- true for current in-progress period
    days_in_period        SMALLINT,
    days_with_data        SMALLINT,  -- may be less than days_in_period for partial

    calculated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(user_id, period_type, period_start)
);
```

**How charts use this table:**
- Monthly Spend Trend bar chart (Trends): `WHERE period_type = 'monthly' ORDER BY period_start DESC LIMIT 6`
- Savings Velocity mini chart (Trends): `savings_rate_pct` per pay period
- Dual bar chart (Income tab): `total_income, total_spend` per monthly snapshot
- Category breakdown (Spend tab, "3 Months" or "12 Months"): aggregate from monthly snapshots

**When "This Month" period is selected:** Current month data is always calculated live from `transactions` (not from snapshots), since the snapshot for the current month won't be complete until month end.

---

## Forecast Cache

### `forecast_cache`
Pre-computed forward-looking projections. Recalculated daily and on-demand when underlying data changes (new recurring bill, goal contribution, etc.). Cached here to avoid expensive projections on every Forecast tab open.

```sql
CREATE TABLE forecast_cache (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Which forecast period this covers
    forecast_month        DATE NOT NULL,  -- first day of the month being forecast
    -- e.g. 2026-03-01 for the March projection

    -- Month-end projection inputs
    expected_income       INTEGER,        -- cents (all income sources for this month)
    actual_income_to_date INTEGER,        -- cents (if partially complete month)
    confirmed_bills_total INTEGER,        -- cents (sum of recurring_bills in this month)
    projected_spend_total INTEGER,        -- cents (bills + projected discretionary)
    projected_surplus     INTEGER,        -- cents (expected_income - projected_spend_total)
    projection_status     VARCHAR(20),    -- 'on_track' | 'at_risk' | 'overspend_risk'

    -- 90-day cash flow (stored as JSONB array of 3 month projections)
    ninety_day_projection JSONB,
    -- e.g. [
    --   {"month": "2026-03", "projected_income": 648000, "projected_spend": 410000, "surplus": 238000},
    --   {"month": "2026-04", ...},
    --   {"month": "2026-05", ...}
    -- ]

    -- Goal ETAs (stored as JSONB array)
    goal_etas             JSONB,
    -- e.g. [
    --   {"goal_id": "uuid", "goal_name": "Japan Trip", "current_amount": 284000,
    --    "target_amount": 420000, "monthly_contribution": 28000, "eta_date": "2026-05-01",
    --    "completion_pct": 67.6},
    --   ...
    -- ]

    -- AI risk flags (stored as JSONB array, ordered by priority score)
    risk_flags            JSONB,
    -- e.g. [
    --   {"flag_type": "spend_risk", "category": "Dining Out", "title": "...", "body": "...",
    --    "priority_score": 2.8, "colour_variant": "red"},
    --   ...
    -- ]

    -- Cache metadata
    calculated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    valid_until           TIMESTAMPTZ,   -- invalidate if underlying data changes before this
    triggered_by          VARCHAR(30),   -- 'scheduled' | 'new_transaction' | 'recurring_bill_change' | 'goal_change'
    is_stale              BOOLEAN DEFAULT FALSE,

    UNIQUE(user_id, forecast_month)
);
```

**Cache invalidation triggers:**
- New transaction inserted → mark current month forecast stale
- Recurring bill added/confirmed/dismissed → mark all future month forecasts stale
- Goal contribution made → mark goal_etas stale
- Daily 06:00 job → regenerate all stale forecasts

**Forecast for current month:**
The `forecast_cache` row for the current month is recalculated on every new transaction (since actual spend affects the projection). Future months are recalculated daily.

---

## Spend Trends Cache

### `spend_trends`
Pre-aggregated period comparisons specifically for the Lifestyle Creep breakdown and the Insights Trends tab. Separate from `metric_snapshots` because these are comparison-window metrics rather than single-period metrics.

```sql
CREATE TABLE spend_trends (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Rolling window end date (typically "today" when calculated)
    as_of_date            DATE NOT NULL,

    -- 3-month rolling averages
    avg_spend_3mo         INTEGER,       -- cents/month
    avg_income_3mo        INTEGER,       -- cents/month

    -- 12-month rolling averages
    avg_spend_12mo        INTEGER,       -- cents/month (NULL if < 12mo data)
    avg_income_12mo       INTEGER,       -- cents/month (NULL if < 12mo data)

    -- Growth rates
    spend_growth_pct      NUMERIC(6,2),  -- (avg_spend_3mo - avg_spend_12mo) / avg_spend_12mo × 100
    income_growth_pct     NUMERIC(6,2),  -- same for income
    creep_delta_pct       NUMERIC(6,2),  -- spend_growth - income_growth

    -- Category-level creep (top driver)
    top_creep_category_id     UUID REFERENCES categories(id),
    top_creep_category_delta  NUMERIC(6,2),  -- % growth for that category (3mo vs 12mo)

    -- Per-category breakdown (JSONB for all categories, not just top)
    category_creep_breakdown  JSONB,
    -- e.g. [
    --   {"category_id": "uuid", "name": "Dining Out", "avg_spend_3mo": 34100,
    --    "avg_spend_12mo": 25400, "growth_pct": 34.3},
    --   ...
    -- ]

    -- Data quality
    months_of_data        SMALLINT,      -- how many months of actual data exist for this user
    has_full_12mo         BOOLEAN,       -- true if 12 months of data available

    calculated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(user_id, as_of_date)  -- one record per user per day
);
```

**Usage:** The Trends tab reads the most recent row for the current user. Calculated daily and on app open if stale. Feeds both the Lifestyle Creep card and the per-category breakdown rows.

---

## Feature → Table Dependency Map

### Spend Tab
| Feature | Primary Tables | Secondary |
|---|---|---|
| KPI row | `transactions` (live) | `pay_periods`, `users` |
| Category breakdown (This Month) | `transactions` (live), `categories` | `budgets` |
| Category breakdown (3M / 12M) | `metric_snapshots`, `categories` | `transactions` (current period live) |
| Merchant analysis | `transactions` (live or snapshots) | — |
| AI insight card | `ai_insights` | — |

### Trends Tab
| Feature | Primary Tables | Secondary |
|---|---|---|
| Lifestyle Creep card | `spend_trends` | `lifestyle_creep_snapshots`, `categories` |
| Safety Net detail | `accounts`, `transactions` | `recurring_bills` |
| Monthly Spend Trend chart | `metric_snapshots` | `transactions` (current month live) |
| Savings Velocity chart | `metric_snapshots`, `pay_periods` | `transactions` |

### Income Tab
| Feature | Primary Tables | Secondary |
|---|---|---|
| KPI row | `transactions` (live income), `pay_periods` | `users` |
| Dual bar chart | `metric_snapshots` | `transactions` (current month live) |
| Pay history | `transactions` (income, by employer) | `users` (income_source_name) |
| Savings breakdown | `savings_goals`, `goal_contributions` | `accounts` |

### Forecast Tab
| Feature | Primary Tables | Secondary |
|---|---|---|
| Month-end projection hero | `forecast_cache` | `transactions`, `recurring_bills`, `pay_periods` |
| 90-day cash flow | `forecast_cache` | `recurring_bills`, `lifestyle_creep_snapshots` |
| Goal ETAs | `forecast_cache`, `goals`, `goal_save_up` | `goal_contributions` |
| Upcoming known expenses | `recurring_bills` | — |
| AI risk flags | `forecast_cache` | `ai_insights`, `budgets`, `savings_goals` |

---

## Indexes

```sql
-- Metric snapshots: chart queries (5 most recent months, by type)
CREATE INDEX idx_metric_snapshots_user_type_period
    ON metric_snapshots(user_id, period_type, period_start DESC);

CREATE INDEX idx_metric_snapshots_partial
    ON metric_snapshots(user_id, period_type, is_partial)
    WHERE is_partial = false;  -- exclude in-progress periods for complete-period charts

-- Forecast cache: current user forecast lookup
CREATE INDEX idx_forecast_cache_user_month
    ON forecast_cache(user_id, forecast_month DESC);

CREATE INDEX idx_forecast_cache_stale
    ON forecast_cache(user_id, is_stale, valid_until)
    WHERE is_stale = true;  -- worker pickup index

-- Spend trends: latest record per user
CREATE INDEX idx_spend_trends_user_date
    ON spend_trends(user_id, as_of_date DESC);
```

---

## Data Flow Notes

### Nightly Snapshot Generation (02:00 AEST)
```
FOR each active user:
  1. Calculate yesterday's completed day data (if applicable to period boundaries)
  2. Check if any monthly periods closed yesterday (end_of_month):
     a. Calculate full month totals from transactions
     b. Insert/update metric_snapshots (period_type = 'monthly', is_partial = false)
  3. Update current month snapshot (period_type = 'monthly', is_partial = true):
     a. Recalculate from transactions WHERE date >= first_day_of_month
     b. Update metric_snapshots row
  4. Recalculate spend_trends (rolling 3mo and 12mo windows)
  5. Regenerate forecast_cache for next 3 months
  6. Update computed_metrics_cache for Insights-specific metrics
```

### On-Demand Recalculation (Triggered Events)
```
New transaction inserted:
  → Mark forecast_cache.is_stale = true for current month
  → Queue metric_snapshot update for current period
  → Do NOT regenerate immediately (defer to next batch or 5-min rolling job)

Recurring bill confirmed/dismissed:
  → Mark forecast_cache.is_stale = true for all future months
  → Queue forecast regeneration

Goal contribution made:
  → Mark forecast_cache.goal_etas as stale
  → Queue goal ETA recalculation

App open (Forecast tab):
  → Check forecast_cache.is_stale for current + next 3 months
  → If stale: regenerate inline (< 200ms target — use cached sub-components)
  → Serve from cache if not stale
```

### Forecast Projection Algorithm (Detailed)
```
For each future month M:

Step 1: Project income
  paydays_in_M = count of expected pay dates within month M
                 (using users.pay_cycle_* to project forward)
  projected_income_M = paydays_in_M × users.expected_net_income
  // If irregular income: use avg_monthly_income from spend_trends

Step 2: Project recurring spend
  confirmed_bills_M = SUM(recurring_bills.amount
                          WHERE next_due_date BETWEEN first(M) AND last(M)
                          AND status = 'confirmed')

  goal_transfers_M = SUM(savings_goals.auto_transfer_amount
                         WHERE next_transfer_date IN M
                         AND auto_transfer_enabled = true)

Step 3: Project discretionary spend
  base = spend_trends.avg_spend_3mo  // 3-month rolling average monthly spend
  base -= confirmed_bills_from_avg   // remove recurring from base (to avoid double-counting)

  // Seasonal adjustment (if 12+ months of data):
  IF spend_trends.has_full_12mo:
    prior_same_month = metric_snapshots WHERE period_start = same month last year
    seasonal_delta = prior_same_month.discretionary_spend - spend_trends.avg_spend_12mo
    base += seasonal_delta × 0.5  // 50% weight to prior year same-month anomaly

  projected_discretionary_M = base

Step 4: Total projection
  projected_spend_M = confirmed_bills_M + goal_transfers_M + projected_discretionary_M
  projected_surplus_M = projected_income_M - projected_spend_M

Step 5: Store in forecast_cache.ninety_day_projection JSONB
```

### Spend Trends Recalculation
```
Runs on:
  - Nightly at 02:00 (full recalculation for all users)
  - App open if last calculated_at > 24 hours ago

Algorithm:
  avg_spend_3mo  = SUM(spend, last 90 days) / 3
  avg_spend_12mo = SUM(spend, last 365 days) / 12  (NULL if < 365 days of data)
  avg_income_3mo  = SUM(income, last 90 days) / 3
  avg_income_12mo = SUM(income, last 365 days) / 12

  spend_growth  = (avg_spend_3mo - avg_spend_12mo) / avg_spend_12mo × 100
  income_growth = (avg_income_3mo - avg_income_12mo) / avg_income_12mo × 100
  creep_delta   = spend_growth - income_growth

  Top creep category:
    FOR each category:
      cat_avg_3mo  = SUM(transactions WHERE category = cat AND date >= today - 90) / 3
      cat_avg_12mo = SUM(transactions WHERE category = cat AND date >= today - 365) / 12
      cat_growth   = (cat_avg_3mo - cat_avg_12mo) / cat_avg_12mo × 100
    top_creep_category = category with max cat_growth (minimum $50/month average to qualify)

  INSERT OR UPDATE spend_trends (user_id, as_of_date = today, ...)
```
