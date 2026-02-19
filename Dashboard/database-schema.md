# Totem — Database Schema
## Dashboard Feature Data Architecture

> **Database:** PostgreSQL (primary). Redis for caching computed metrics.
> **Conventions:** All tables use `UUID` primary keys. Timestamps are `TIMESTAMPTZ` (timezone-aware). Soft deletes via `deleted_at` where noted. Amounts stored as `INTEGER` in cents to avoid floating-point rounding errors (e.g. $18.99 → `1899`).

---

## Table of Contents
1. [Core Tables](#core-tables)
2. [Transaction & Categorisation](#transaction--categorisation)
3. [Pay Periods & Income](#pay-periods--income)
4. [Budgets & Goals](#budgets--goals)
5. [Recurring Bills](#recurring-bills)
6. [AI & Insights](#ai--insights)
7. [Snapshots & Computed Metrics](#snapshots--computed-metrics)
8. [Feature → Table Dependency Map](#feature--table-dependency-map)
9. [Indexes](#indexes)
10. [Data Flow Notes](#data-flow-notes)

---

## Core Tables

### `users`
```sql
CREATE TABLE users (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email                 VARCHAR(255) UNIQUE NOT NULL,
    full_name             VARCHAR(255),
    first_name            VARCHAR(100),
    currency              CHAR(3) NOT NULL DEFAULT 'AUD',  -- 'AUD' or 'NZD'
    timezone              VARCHAR(50) NOT NULL DEFAULT 'Australia/Sydney',
    country               CHAR(2) NOT NULL DEFAULT 'AU',   -- 'AU' or 'NZ'

    -- Pay cycle (populated during onboarding or via detection)
    pay_cycle_type        VARCHAR(20),  -- 'weekly' | 'fortnightly' | 'monthly' | 'irregular'
    pay_cycle_day_of_week SMALLINT,     -- 0=Mon..6=Sun (for weekly/fortnightly)
    pay_cycle_day_of_month SMALLINT,   -- 1–31 (for monthly)
    pay_cycle_length_days  SMALLINT,   -- derived: 7, 14, or 28-31
    income_source_name    VARCHAR(255), -- employer name as it appears on bank feed
    expected_net_income   INTEGER,      -- cents, per pay period

    -- Income type
    income_is_irregular   BOOLEAN NOT NULL DEFAULT FALSE,

    -- CSV import tracking
    csv_import_earliest_date DATE,      -- earliest transaction date from any CSV import
    csv_import_latest_date   DATE,

    -- Onboarding
    onboarding_complete   BOOLEAN NOT NULL DEFAULT FALSE,
    onboarding_step       VARCHAR(50),   -- current step if incomplete

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at            TIMESTAMPTZ     -- soft delete
);
```

---

### `accounts`
One row per linked bank account. Populated and refreshed via Basiq (AU) / equivalent NZ connector.

```sql
CREATE TABLE accounts (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- External connector
    basiq_account_id      VARCHAR(255) UNIQUE,  -- Basiq's account reference
    connector_id          VARCHAR(255),          -- which connector (Basiq AU, Akahu NZ, etc.)

    -- Account metadata
    institution_name      VARCHAR(255),          -- e.g. "ANZ", "Westpac"
    account_name          VARCHAR(255),          -- e.g. "Everyday Account"
    account_number_masked VARCHAR(20),           -- last 4 digits only
    account_type          VARCHAR(30) NOT NULL,
        -- 'transaction' | 'savings' | 'credit_card' | 'loan' | 'mortgage'
        -- | 'investment' | 'superannuation' | 'term_deposit'

    -- Balances (in cents)
    balance               INTEGER NOT NULL DEFAULT 0,
    available_balance     INTEGER,               -- may differ from balance for credit
    credit_limit          INTEGER,               -- for credit cards

    -- Flags
    is_primary_transaction BOOLEAN DEFAULT FALSE, -- the account salary is paid into
    include_in_net_worth  BOOLEAN DEFAULT TRUE,
    include_in_safety_net BOOLEAN DEFAULT TRUE,   -- false for investments, super
    is_active             BOOLEAN DEFAULT TRUE,

    last_synced_at        TIMESTAMPTZ,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Transaction & Categorisation

### `transactions`
```sql
CREATE TABLE transactions (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    account_id            UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,

    -- External reference
    basiq_transaction_id  VARCHAR(255) UNIQUE,
    source                VARCHAR(20) NOT NULL DEFAULT 'bank_feed',
        -- 'bank_feed' | 'csv_import' | 'manual'

    -- Core fields
    amount                INTEGER NOT NULL,    -- positive = debit (spend), negative = credit (income)
    description           TEXT,               -- raw bank description
    merchant_name         VARCHAR(255),       -- cleaned merchant name
    merchant_name_normalised VARCHAR(255),    -- further normalised for grouping (strip PTY LTD etc.)
    merchant_category_code VARCHAR(10),       -- MCC from bank/Basiq
    date                  DATE NOT NULL,      -- transaction date (not posted date)
    posted_date           DATE,

    -- Classification
    is_income             BOOLEAN NOT NULL DEFAULT FALSE,
    is_transfer           BOOLEAN NOT NULL DEFAULT FALSE,
    is_recurring          BOOLEAN NOT NULL DEFAULT FALSE,

    -- Category assignment
    category_id           UUID REFERENCES categories(id),
    category_assigned_by  VARCHAR(10) DEFAULT 'ai',  -- 'ai' | 'user' | 'rule'
    category_confidence   NUMERIC(3,2),              -- 0.00–1.00 (AI confidence score)

    -- Recurring bill link
    recurring_bill_id     UUID REFERENCES recurring_bills(id),

    -- Pay period link
    pay_period_id         UUID REFERENCES pay_periods(id),

    -- Notes / tags
    notes                 TEXT,
    user_tags             VARCHAR(50)[],       -- user-applied tags

    -- Deduplication
    is_duplicate          BOOLEAN DEFAULT FALSE,
    duplicate_of          UUID REFERENCES transactions(id),

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `categories`
System-defined defaults + user-created custom categories.

```sql
CREATE TABLE categories (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID REFERENCES users(id) ON DELETE CASCADE,
        -- NULL = system default category (shared across all users)

    name                  VARCHAR(100) NOT NULL,
    slug                  VARCHAR(100) NOT NULL,     -- e.g. 'eating-out', 'groceries'
    icon                  VARCHAR(10),               -- emoji or icon name
    colour                VARCHAR(7),                -- hex colour for charts
    parent_category_id    UUID REFERENCES categories(id),  -- for subcategories

    -- Flags
    is_system             BOOLEAN NOT NULL DEFAULT FALSE,
    is_subscription       BOOLEAN NOT NULL DEFAULT FALSE,  -- feeds Subscription Load
    is_income_category    BOOLEAN NOT NULL DEFAULT FALSE,
    is_transfer_category  BOOLEAN NOT NULL DEFAULT FALSE,
    is_active             BOOLEAN NOT NULL DEFAULT TRUE,
    sort_order            SMALLINT DEFAULT 0,

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(user_id, slug)  -- user can't have two categories with same slug
);
```

**System category seeds (examples):**
| Slug | Name | Icon | Is Subscription |
|---|---|---|---|
| housing | Housing / Rent | 🏠 | false |
| groceries | Groceries | 🛒 | false |
| eating-out | Eating Out | 🍔 | false |
| transport | Transport | 🚗 | false |
| entertainment | Entertainment | 🎭 | false |
| subscriptions | Subscriptions | 📱 | true |
| health | Health & Medical | 💊 | false |
| personal-care | Personal Care | 💅 | false |
| shopping | Shopping | 🛍️ | false |
| utilities | Utilities | 💡 | false |
| insurance | Insurance | 🛡️ | false |
| savings-transfer | Savings Transfer | 💰 | false (transfer) |
| income | Income / Salary | 💵 | false (income) |

---

### `categorisation_rules`
User-defined or AI-suggested rules that auto-assign categories to future transactions.

```sql
CREATE TABLE categorisation_rules (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Match condition
    match_type            VARCHAR(20) NOT NULL,  -- 'merchant_name' | 'description_contains' | 'amount_range'
    match_value           VARCHAR(255) NOT NULL, -- value to match against
    match_is_regex        BOOLEAN DEFAULT FALSE,

    -- Assignment
    category_id           UUID NOT NULL REFERENCES categories(id),

    -- Meta
    created_by            VARCHAR(10) DEFAULT 'user',  -- 'user' | 'ai_suggestion'
    is_active             BOOLEAN DEFAULT TRUE,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Pay Periods & Income

### `pay_periods`
Each row represents one pay cycle for a user.

```sql
CREATE TABLE pay_periods (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    start_date            DATE NOT NULL,
    end_date              DATE NOT NULL,
    length_days           SMALLINT NOT NULL,  -- end_date - start_date

    -- Income
    expected_income       INTEGER,     -- cents, set from users.expected_net_income at period creation
    actual_income         INTEGER,     -- cents, populated when income transaction detected

    -- Status
    status                VARCHAR(20) NOT NULL DEFAULT 'upcoming',
        -- 'upcoming' | 'active' | 'completed'

    -- Computed values (cached for performance)
    total_spend           INTEGER,     -- cents, recalculated on each new transaction
    total_discretionary_spend INTEGER,
    savings_rate_pct      NUMERIC(5,2),  -- calculated on period close

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(user_id, start_date)
);
```

---

## Budgets & Goals

### `budgets`
User-set spending limits per category per period.

```sql
CREATE TABLE budgets (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    category_id           UUID NOT NULL REFERENCES categories(id),

    amount                INTEGER NOT NULL,   -- cents
    period_type           VARCHAR(20) NOT NULL DEFAULT 'monthly',
        -- 'weekly' | 'fortnightly' | 'monthly'

    is_active             BOOLEAN NOT NULL DEFAULT TRUE,
    start_date            DATE,    -- when this budget became active (NULL = always)
    end_date              DATE,    -- if budget is time-limited (NULL = ongoing)

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(user_id, category_id)  -- one active budget per category
);
```

---

### `savings_goals`
User-defined savings targets with optional auto-transfer rules.

```sql
CREATE TABLE savings_goals (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    name                  VARCHAR(255) NOT NULL,  -- e.g. "House Deposit", "Europe Trip"
    icon                  VARCHAR(10),
    colour                VARCHAR(7),

    -- Target
    target_amount         INTEGER NOT NULL,        -- cents
    current_amount        INTEGER NOT NULL DEFAULT 0,  -- cents (updated on transfers)
    target_date           DATE,

    -- Auto-transfer
    auto_transfer_enabled BOOLEAN DEFAULT FALSE,
    auto_transfer_amount  INTEGER,                -- cents per transfer
    auto_transfer_frequency VARCHAR(20),          -- 'weekly' | 'fortnightly' | 'monthly'
    next_transfer_date    DATE,                   -- feeds Safe to Spend calculation

    -- Linked account
    savings_account_id    UUID REFERENCES accounts(id),  -- where the money is held

    status                VARCHAR(20) DEFAULT 'active',  -- 'active' | 'completed' | 'paused'

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at          TIMESTAMPTZ
);
```

---

## Recurring Bills

### `recurring_bills`
Detected or manually added recurring charges.

```sql
CREATE TABLE recurring_bills (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Identity
    merchant_name         VARCHAR(255) NOT NULL,
    merchant_name_normalised VARCHAR(255),
    category_id           UUID REFERENCES categories(id),

    -- Amount
    amount                INTEGER NOT NULL,          -- cents
    amount_variance_pct   NUMERIC(4,2),              -- how much amount varies (e.g. 0.05 = ±5%)

    -- Schedule
    frequency             VARCHAR(20) NOT NULL,
        -- 'weekly' | 'fortnightly' | 'monthly' | 'quarterly' | 'annual'
    average_interval_days NUMERIC(5,1),             -- actual mean interval from detection
    interval_stddev_days  NUMERIC(4,1),             -- standard deviation

    -- Dates
    first_detected_date   DATE,
    last_charged_date     DATE NOT NULL,
    next_due_date         DATE NOT NULL,

    -- Approval state
    status                VARCHAR(30) NOT NULL DEFAULT 'pending_approval',
        -- 'pending_approval' | 'confirmed' | 'dismissed' | 'cancelled'
    detection_confidence  VARCHAR(10),              -- 'high' | 'medium' | 'low'
    confirmed_by_user     BOOLEAN DEFAULT FALSE,
    confirmed_at          TIMESTAMPTZ,
    dismissed_at          TIMESTAMPTZ,
    dismissal_reason      VARCHAR(100),

    -- Flags
    is_subscription       BOOLEAN DEFAULT FALSE,    -- feeds Subscription Load widget
    last_used_check_date  DATE,                     -- for unused subscription detection

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## AI & Insights

### `ai_insights`
Each row is one generated insight for a user.

```sql
CREATE TABLE ai_insights (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Insight classification
    insight_type          VARCHAR(50) NOT NULL,
        -- 'category_over_budget' | 'category_at_risk' | 'unused_subscription'
        -- | 'unusual_transaction' | 'low_balance_before_payday'
        -- | 'safety_net_low' | 'goal_at_risk' | 'positive_savings_milestone'
        -- | 'positive_under_budget' | 'lifestyle_creep_detected'

    -- Display
    icon                  VARCHAR(10),             -- emoji for insight card
    tag_text              VARCHAR(60),             -- e.g. "⚠ Spending Alert"
    body_text             TEXT NOT NULL,           -- e.g. "Eating out up 67% vs last month..."
    cta_label             VARCHAR(60),             -- e.g. "See breakdown"
    cta_action            VARCHAR(50),             -- e.g. 'navigate:categories/eating-out'

    -- Card border colour
    card_variant          VARCHAR(20) DEFAULT 'warning',
        -- 'warning' (amber) | 'urgent' (red) | 'positive' (green) | 'info' (brand)

    -- Scoring
    priority_score        NUMERIC(5,2) NOT NULL,
    severity_score        SMALLINT,                -- 0–3
    urgency_score         SMALLINT,                -- 0–3
    dollar_impact         INTEGER,                 -- cents

    -- Related data references (for "See breakdown" navigation)
    related_entity_type   VARCHAR(30),             -- 'category' | 'transaction' | 'recurring_bill' | 'goal'
    related_entity_id     UUID,

    -- State
    generated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    shown_at              TIMESTAMPTZ,
    dismissed_at          TIMESTAMPTZ,
    acted_on_at           TIMESTAMPTZ,
    expires_at            TIMESTAMPTZ,             -- insight becomes stale after this

    -- Deduplication — don't generate same insight type twice within window
    dedup_key             VARCHAR(255),            -- e.g. 'category_over_budget:eating-out:2026-02'
    UNIQUE(user_id, dedup_key)
);
```

---

### `notification_log`
Track every push notification sent (for the AI proactive notification system).

```sql
CREATE TABLE notification_log (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    ai_insight_id         UUID REFERENCES ai_insights(id),

    notification_type     VARCHAR(50) NOT NULL,
        -- 'proactive_nudge' | 'bill_reminder' | 'budget_alert' | 'payday_approaching'
        -- | 'goal_milestone' | 'unusual_transaction' | 'low_balance'

    title                 VARCHAR(255),
    body                  TEXT,
    payload               JSONB,                   -- any deep-link or action data

    -- Delivery
    sent_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    delivered_at          TIMESTAMPTZ,
    opened_at             TIMESTAMPTZ,
    acted_on_at           TIMESTAMPTZ,

    -- AI learning (for improving notification relevance over time)
    was_acted_on          BOOLEAN DEFAULT FALSE,
    user_feedback         VARCHAR(20),             -- 'helpful' | 'not_helpful' | 'dismissed'
    feedback_at           TIMESTAMPTZ
);
```

---

## Snapshots & Computed Metrics

### `lifestyle_creep_snapshots`
Monthly snapshots used to power the Lifestyle Creep metric and chart.

```sql
CREATE TABLE lifestyle_creep_snapshots (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    snapshot_month        DATE NOT NULL,            -- first day of the month this represents
    -- e.g. 2026-02-01 represents February 2026

    -- Spending
    total_spend_this_month  INTEGER,               -- cents
    avg_spend_3mo           INTEGER,               -- cents (3-month rolling at this point)
    avg_spend_12mo          INTEGER,               -- cents (12-month rolling, NULL if < 12mo data)

    -- Income
    total_income_this_month INTEGER,
    avg_income_3mo          INTEGER,
    avg_income_12mo         INTEGER,

    -- Computed ratios
    spend_growth_3v12_pct   NUMERIC(6,2),          -- % change 3mo vs 12mo avg spend
    income_growth_3v12_pct  NUMERIC(6,2),
    creep_delta_pct         NUMERIC(6,2),          -- spend_growth - income_growth
    creep_status            VARCHAR(20),
        -- 'improving' | 'balanced' | 'watch' | 'detected'

    -- Data quality
    months_of_data          SMALLINT,              -- how many months of data existed at snapshot time
    is_complete_data        BOOLEAN,               -- TRUE if full 12 months available

    calculated_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(user_id, snapshot_month)
);
```

---

### `computed_metrics_cache`
Redis is primary cache, but this table provides a persistent fallback and audit trail for dashboard-critical computed values.

```sql
CREATE TABLE computed_metrics_cache (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    metric_key            VARCHAR(100) NOT NULL,
        -- 'safe_to_spend' | 'month_end_projection' | 'safety_net_months'
        -- | 'savings_velocity_pct' | 'subscription_monthly_cost' | 'subscription_count'
        -- | 'lifestyle_creep_status' | 'current_pay_period_id'

    value_integer         INTEGER,                -- for amounts in cents
    value_numeric         NUMERIC(10,4),          -- for percentages, ratios
    value_text            VARCHAR(255),           -- for status strings
    value_json            JSONB,                  -- for complex cached values

    computed_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at            TIMESTAMPTZ,            -- when to recompute
    is_stale              BOOLEAN DEFAULT FALSE,

    UNIQUE(user_id, metric_key)
);
```

---

### `user_settings`
Flexible key-value store for user preferences.

```sql
CREATE TABLE user_settings (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    setting_key           VARCHAR(100) NOT NULL,
    setting_value         JSONB NOT NULL,

    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(user_id, setting_key)
);
```

**Common setting keys:**
| Key | Type | Default | Purpose |
|---|---|---|---|
| `dashboard_tier_unlocks` | object | `{tier2: false, tier3: false}` | Controls tier visibility |
| `safety_net_target_months` | integer | `3` | User's self-set target for safety net |
| `push_notifications_enabled` | boolean | `true` | Global notification toggle |
| `notification_quiet_hours` | object | `{start: 22, end: 7}` | Don't send between these hours |
| `pay_cycle_override` | object | null | Manual pay cycle if detection wrong |
| `irregular_income_acknowledged` | boolean | false | User confirmed they have irregular income |
| `csv_import_completed` | boolean | false | Onboarding CSV import done |

---

## Feature → Table Dependency Map

| Dashboard Feature | Primary Tables | Secondary / Supporting |
|---|---|---|
| **Safe to Spend** | `pay_periods`, `transactions`, `recurring_bills`, `savings_goals` | `accounts`, `users` |
| **Month End Projection** | `transactions`, `recurring_bills`, `pay_periods` | `users` (pay cycle) |
| **Payday Anchor** | `pay_periods`, `users` | `transactions` (for detection) |
| **AI Insight Card** | `ai_insights` | `transactions`, `budgets`, `recurring_bills`, `savings_goals`, `accounts` |
| **Upcoming Bills** | `recurring_bills` | — |
| **Budget Burn Rate** | `budgets`, `transactions`, `categories` | `pay_periods` |
| **Subscription Load** | `recurring_bills` | `transactions` (last used check) |
| **Safety Net** | `accounts`, `transactions` | `recurring_bills` (committed next month) |
| **Savings Velocity** | `pay_periods`, `transactions` | `savings_goals` |
| **Lifestyle Creep** | `lifestyle_creep_snapshots` | `transactions` (for current month live) |
| **Category % of Income** | `transactions`, `categories` | `pay_periods` (income figure) |
| **Recurring Bill Detection** | `transactions` | `recurring_bills` (write target) |
| **Pay Cycle Detection** | `transactions` | `users` (write target), `pay_periods` (create target) |

---

## Indexes

```sql
-- Transactions: the most queried table
CREATE INDEX idx_transactions_user_date        ON transactions(user_id, date DESC);
CREATE INDEX idx_transactions_user_period      ON transactions(user_id, pay_period_id);
CREATE INDEX idx_transactions_user_category    ON transactions(user_id, category_id);
CREATE INDEX idx_transactions_user_recurring   ON transactions(user_id, is_recurring, recurring_bill_id);
CREATE INDEX idx_transactions_user_income      ON transactions(user_id, is_income, date DESC);
CREATE INDEX idx_transactions_merchant         ON transactions(user_id, merchant_name_normalised);
CREATE INDEX idx_transactions_source           ON transactions(source);  -- for CSV import filtering

-- Recurring bills
CREATE INDEX idx_recurring_bills_user_status   ON recurring_bills(user_id, status);
CREATE INDEX idx_recurring_bills_next_due      ON recurring_bills(user_id, next_due_date);
CREATE INDEX idx_recurring_bills_subscription  ON recurring_bills(user_id, is_subscription, status);

-- Pay periods
CREATE INDEX idx_pay_periods_user_status       ON pay_periods(user_id, status);
CREATE INDEX idx_pay_periods_user_dates        ON pay_periods(user_id, start_date, end_date);

-- AI insights
CREATE INDEX idx_ai_insights_user_priority     ON ai_insights(user_id, priority_score DESC, dismissed_at NULLS FIRST);
CREATE INDEX idx_ai_insights_dedup             ON ai_insights(user_id, dedup_key);
CREATE INDEX idx_ai_insights_unseen            ON ai_insights(user_id, shown_at) WHERE shown_at IS NULL;

-- Budgets
CREATE INDEX idx_budgets_user_active           ON budgets(user_id, is_active, category_id);

-- Lifestyle creep
CREATE INDEX idx_lifestyle_snapshots_user      ON lifestyle_creep_snapshots(user_id, snapshot_month DESC);

-- Computed cache
CREATE INDEX idx_computed_cache_user_key       ON computed_metrics_cache(user_id, metric_key);

-- Accounts
CREATE INDEX idx_accounts_user_type           ON accounts(user_id, account_type, is_active);
```

---

## Data Flow Notes

### Basiq Webhook → Transaction → Metric Update
```
1. Basiq fires webhook: new transaction posted
2. API receives webhook → validate → insert into transactions table
3. AI categorisation job fires (async): classify transaction → update category_id
4. Recurring bill detection job fires (async): check if this transaction matches any recurring pattern
5. Pay detection job fires (async): if is_income = true → check if matches pay cycle
6. Metric recalculation fires (async):
   - Recompute safe_to_spend
   - Recompute month_end_projection
   - Recompute safety_net_months
   - Check if any new insights should be generated
   - Update computed_metrics_cache
   - Invalidate Redis cache keys for this user
7. Push notification check: if new high-priority insight generated → check quiet hours → send push
```

### Amount Storage Convention
```
All monetary amounts stored as INTEGER in cents.
$18.99  → 1899
$3,200  → 320000
-$45.00 → -4500  (for income/credit transactions)

Display formatting is handled in the frontend:
amount_in_cents / 100 → formatted with locale-appropriate currency symbol
```

### CSV Import Flow
```
1. User uploads CSV on onboarding
2. Parse CSV → map columns to transaction schema
3. Insert transactions with source = 'csv_import'
4. Update users.csv_import_earliest_date and csv_import_latest_date
5. Run categorisation on all imported transactions (batch AI job)
6. Run recurring bill detection on imported history
7. Run lifestyle_creep_snapshots generation for all historical months
8. Update onboarding_step → 'complete'
```

### Pay Period Auto-Generation
```
When pay cycle is confirmed (either detected or manually set):
1. Create current active pay_period row
2. Create next upcoming pay_period row
3. Link all existing transactions that fall within each period via pay_period_id
4. Schedule a job to create the next pay_period 3 days before current one ends
```

### Lifestyle Creep Snapshot Schedule
```
Runs on 1st of each month at 02:00 AEST/NZST:
1. For each active user:
   - Calculate avg_spend_3mo, avg_spend_12mo, avg_income_3mo, avg_income_12mo
   - Compute creep_delta_pct and creep_status
   - Insert into lifestyle_creep_snapshots
   - If creep_status changed from prior month → generate lifestyle_creep_detected insight
```
