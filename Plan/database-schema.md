# Totem — Plan Screen Database Schema
## Goals + Budgets Data Architecture

> **Important:** The `savings_goals` table defined in `Dashboard/database-schema.md` is **superseded** by the comprehensive goals schema below. The `budgets` table from the Dashboard schema remains unchanged — it is reused here without modification.
> All amounts in INTEGER cents. All timestamps TIMESTAMPTZ.

---

## Goals Schema

### `goals` (base table — all goal types)
```sql
CREATE TABLE goals (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Type and identity
    goal_type             VARCHAR(20) NOT NULL,
        -- 'save_up' | 'spending_habit' | 'debt_paydown'
    name                  VARCHAR(255) NOT NULL,
    icon                  VARCHAR(10),                -- emoji
    colour                VARCHAR(7),                 -- hex, user-selected or default by type
    motivation_text       TEXT,                       -- "why is this goal important to you?"

    -- AI provenance
    is_ai_generated       BOOLEAN NOT NULL DEFAULT FALSE,
    ai_suggestion_id      UUID REFERENCES ai_goal_suggestions(id),
    ai_baseline_context   JSONB,
        -- stores the data the AI used to generate this goal:
        -- { "baseline_months": 3, "avg_spend": 58000, "reduction_pct": 20 }

    -- State
    status                VARCHAR(20) NOT NULL DEFAULT 'active',
        -- 'active' | 'completed' | 'paused' | 'archived'

    -- Notification preferences (per goal)
    notify_on_track       BOOLEAN DEFAULT TRUE,
    notify_at_risk        BOOLEAN DEFAULT TRUE,
    notify_completion     BOOLEAN DEFAULT TRUE,
    notify_streak_milestones BOOLEAN DEFAULT TRUE,

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at          TIMESTAMPTZ,
    archived_at           TIMESTAMPTZ
);
```

---

### `goal_save_up`
Type-specific data for Save Up goals.

```sql
CREATE TABLE goal_save_up (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goal_id               UUID NOT NULL UNIQUE REFERENCES goals(id) ON DELETE CASCADE,

    -- Target
    target_amount         INTEGER NOT NULL,           -- cents
    target_date           DATE,                       -- optional deadline

    -- Current state
    current_amount        INTEGER NOT NULL DEFAULT 0, -- cents
    -- Derived: progress_pct = current_amount / target_amount × 100

    -- Contributions
    monthly_contribution  INTEGER,                    -- cents, user's intended monthly amount
    auto_transfer_enabled BOOLEAN DEFAULT FALSE,
    auto_transfer_day     SMALLINT,                   -- day of month (1-28)
    next_transfer_date    DATE,

    -- Linked account
    savings_account_id    UUID REFERENCES accounts(id),

    -- Behaviour flag
    spending_reduces_progress BOOLEAN DEFAULT FALSE,
        -- TRUE for emergency funds (spending from the account reduces goal progress)
        -- FALSE for sinking funds where you spend the whole amount (vacation)

    -- Cached projections (recomputed on each contribution)
    avg_monthly_contribution_cents INTEGER,           -- rolling 3-month avg
    projected_completion_date DATE,                   -- at current rate
    projected_status      VARCHAR(20),               -- 'ahead' | 'on_track' | 'at_risk' | 'behind'

    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `goal_spending_habit`
Type-specific data for Spending Habit goals. Totem's signature goal type.

```sql
CREATE TABLE goal_spending_habit (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goal_id               UUID NOT NULL UNIQUE REFERENCES goals(id) ON DELETE CASCADE,

    -- Category being tracked
    category_id           UUID NOT NULL REFERENCES categories(id),

    -- Baseline (captured at goal creation, immutable)
    baseline_amount       INTEGER NOT NULL,           -- cents/month (3-month avg at creation)
    baseline_period_months SMALLINT DEFAULT 3,
    baseline_calculated_at DATE NOT NULL,

    -- Target
    reduction_pct         NUMERIC(5,2) NOT NULL,      -- e.g. 20.00 = 20%
    target_amount         INTEGER NOT NULL,            -- cents/month = baseline × (1 - reduction_pct/100)

    -- Period config
    period_type           VARCHAR(20) NOT NULL DEFAULT 'monthly',
    auto_renew            BOOLEAN NOT NULL DEFAULT TRUE,

    -- Streak (updated on each month close)
    current_streak        INTEGER NOT NULL DEFAULT 0,
    longest_streak        INTEGER NOT NULL DEFAULT 0,
    total_periods_hit     INTEGER NOT NULL DEFAULT 0,
    total_periods_tracked INTEGER NOT NULL DEFAULT 0,

    -- Budget link (optional)
    linked_budget_id      UUID REFERENCES budgets(id),
        -- if set: budget.amount stays in sync with target_amount

    -- High-risk day tracking (updated weekly)
    high_risk_days        SMALLINT[],                 -- day-of-week array: [4] = Thursday
    high_risk_hours       SMALLINT[],                 -- hour-of-day array: [18, 19, 20]
    last_pattern_scan_at  TIMESTAMPTZ,

    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `goal_debt_paydown`
Type-specific data for Debt Paydown goals.

```sql
CREATE TABLE goal_debt_paydown (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goal_id               UUID NOT NULL UNIQUE REFERENCES goals(id) ON DELETE CASCADE,

    -- Linked account
    account_id            UUID NOT NULL REFERENCES accounts(id),
        -- must be account_type IN ('credit_card', 'loan', 'mortgage')

    -- Amounts
    starting_balance      INTEGER NOT NULL,           -- cents, balance when goal was created
    target_balance        INTEGER NOT NULL DEFAULT 0, -- cents, usually 0 (fully paid off)
    monthly_payment       INTEGER,                    -- cents, user's intended monthly payment

    -- Optional
    target_date           DATE,
    interest_rate         NUMERIC(5,2),               -- annual %, for interest saved calculation

    -- Cached projections
    avg_monthly_payment_cents INTEGER,               -- rolling 3-month avg of actual payments
    projected_payoff_date DATE,
    projected_status      VARCHAR(20),

    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `goal_period_records`
Monthly performance history for Spending Habit goals. One row per goal per month.

```sql
CREATE TABLE goal_period_records (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goal_id               UUID NOT NULL REFERENCES goals(id) ON DELETE CASCADE,

    -- Period
    period_start          DATE NOT NULL,              -- first day of the month
    period_end            DATE NOT NULL,              -- last day of the month

    -- Amounts (in cents)
    target_amount         INTEGER NOT NULL,           -- what the goal was set to for this period
    actual_amount         INTEGER NOT NULL,           -- what was actually spent

    -- Result
    was_successful        BOOLEAN NOT NULL,           -- actual_amount <= target_amount
    variance_amount       INTEGER,                    -- actual - target (negative = under = good)
    variance_pct          NUMERIC(6,2),              -- (actual - target) / target × 100

    -- Streak state at period close
    streak_at_close       INTEGER NOT NULL DEFAULT 0,

    -- Celebration
    celebration_shown_at  TIMESTAMPTZ,               -- when the success animation was shown

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(goal_id, period_start)
);
```

---

### `ai_goal_suggestions`
AI-generated goal recommendations pending user review.

```sql
CREATE TABLE ai_goal_suggestions (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Suggested goal details
    goal_type             VARCHAR(20) NOT NULL,
    suggested_name        VARCHAR(255) NOT NULL,
    suggested_icon        VARCHAR(10),
    insight_text          TEXT NOT NULL,             -- "Your eating out spend has risen 40% in 3 months"
    preview_text          TEXT,                      -- "Based on $580/mo avg → $464/mo target (-20%)"

    -- Pre-filled goal data (for the review sheet)
    prefilled_data        JSONB NOT NULL,
        -- For spending_habit: { category_id, baseline_amount, target_amount, reduction_pct }
        -- For save_up: { target_amount, monthly_contribution, projected_date }
        -- For debt_paydown: { account_id, starting_balance, monthly_payment }

    -- Priority (same scoring as AI Insights)
    priority_score        NUMERIC(5,2) NOT NULL,
    trigger_type          VARCHAR(50),
        -- 'category_overspend_trend' | 'surplus_detected' | 'high_utilization'
        -- | 'subscription_overload' | 'user_chat_request' | 'milestone_reached'

    -- State
    status                VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- 'pending' | 'accepted' (goal created) | 'snoozed' | 'dismissed'
    snoozed_until         TIMESTAMPTZ,
    accepted_goal_id      UUID REFERENCES goals(id),  -- set when user creates the goal

    generated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    shown_at              TIMESTAMPTZ,
    acted_on_at           TIMESTAMPTZ,

    -- Dedup
    dedup_key             VARCHAR(255),
    UNIQUE(user_id, dedup_key)
);
```

---

### `goal_contributions`
Audit trail for every amount added to a Save Up goal.

```sql
CREATE TABLE goal_contributions (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goal_id               UUID NOT NULL REFERENCES goals(id) ON DELETE CASCADE,
    user_id               UUID NOT NULL REFERENCES users(id),

    amount                INTEGER NOT NULL,           -- cents
    source                VARCHAR(20) NOT NULL,
        -- 'auto_transfer' | 'manual' | 'detected_transfer'

    -- For detected_transfer: link to the transaction
    transaction_id        UUID REFERENCES transactions(id),

    -- Celebration state
    celebrated            BOOLEAN DEFAULT FALSE,
    celebrated_at         TIMESTAMPTZ,

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `user_ai_activation`
Tracks the 30-transaction milestone and overall AI activation state.

```sql
CREATE TABLE user_ai_activation (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,

    -- Milestone tracking
    reviewed_transaction_count INTEGER NOT NULL DEFAULT 0,
    milestone_reached         BOOLEAN NOT NULL DEFAULT FALSE,
    milestone_reached_at      TIMESTAMPTZ,
    milestone_celebrated      BOOLEAN DEFAULT FALSE,

    -- AI active state
    ai_goal_suggestions_active BOOLEAN DEFAULT FALSE,
    ai_insights_active         BOOLEAN DEFAULT FALSE,  -- from Dashboard
    ai_pattern_detection_active BOOLEAN DEFAULT FALSE,

    -- Overall AI feature activation date
    activated_at              TIMESTAMPTZ,

    updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Reused from Dashboard Schema (unchanged)

### `budgets` (no changes needed)
The budgets table from `Dashboard/database-schema.md` is used as-is. The Budgets sub-tab reads and writes to this table.

Key fields relevant to Plan screen:
- `user_id`, `category_id`, `amount` (monthly limit in cents)
- `period_type` (monthly)
- `is_active`

The only addition is an optional foreign key for goal-budget sync:

```sql
-- Migration: add to existing budgets table
ALTER TABLE budgets ADD COLUMN linked_goal_id UUID REFERENCES goals(id) ON DELETE SET NULL;
-- When linked: budget.amount stays synced with goal_spending_habit.target_amount
```

---

## Indexes

```sql
-- Goals: primary queries
CREATE INDEX idx_goals_user_type_status ON goals(user_id, goal_type, status);
CREATE INDEX idx_goals_user_active      ON goals(user_id, status) WHERE status = 'active';

-- Goal type lookups
CREATE INDEX idx_goal_save_up_goal           ON goal_save_up(goal_id);
CREATE INDEX idx_goal_spending_habit_goal    ON goal_spending_habit(goal_id);
CREATE INDEX idx_goal_spending_habit_category ON goal_spending_habit(category_id);
CREATE INDEX idx_goal_debt_paydown_account   ON goal_debt_paydown(account_id);

-- Period records: most queried for history/streak
CREATE INDEX idx_goal_period_records_goal_date ON goal_period_records(goal_id, period_start DESC);
CREATE INDEX idx_goal_period_records_success   ON goal_period_records(goal_id, was_successful);

-- AI suggestions
CREATE INDEX idx_ai_goal_suggestions_user_status ON ai_goal_suggestions(user_id, status);
CREATE INDEX idx_ai_goal_suggestions_pending     ON ai_goal_suggestions(user_id, priority_score DESC)
    WHERE status = 'pending';

-- Contributions
CREATE INDEX idx_goal_contributions_goal   ON goal_contributions(goal_id, created_at DESC);

-- AI activation
CREATE INDEX idx_user_ai_activation_user   ON user_ai_activation(user_id);
```

---

## Feature → Table Dependency Map

| Feature | Primary Tables | Secondary |
|---|---|---|
| **Save Up goal** | `goals`, `goal_save_up` | `accounts`, `transactions`, `goal_contributions` |
| **Spending Habit goal** | `goals`, `goal_spending_habit` | `transactions`, `categories`, `goal_period_records` |
| **Debt Paydown goal** | `goals`, `goal_debt_paydown` | `accounts`, `transactions` |
| **Goal streaks** | `goal_spending_habit` | `goal_period_records` |
| **AI goal suggestions** | `ai_goal_suggestions` | `goals` (on acceptance) |
| **30-transaction milestone** | `user_ai_activation` | `transactions` (reviewed count) |
| **Budgets tab** | `budgets` | `transactions`, `categories` |
| **Budget-goal sync** | `budgets.linked_goal_id`, `goal_spending_habit` | — |
| **High-risk day notifications** | `goal_spending_habit.high_risk_days` | `transactions` (pattern detection) |
| **Month close / streak update** | `goal_period_records`, `goal_spending_habit` | `notification_log` |

---

## Data Flow Notes

### Goal Creation (AI-generated path)
```
1. AI suggestion generated → inserted into ai_goal_suggestions (status: pending)
2. Suggestion card shown in Goals tab
3. User taps "Review Suggestion" → goal creation sheet slides up, pre-filled
4. User taps "Create Goal" →
   a. INSERT into goals (base)
   b. INSERT into goal_spending_habit / goal_save_up / goal_debt_paydown
   c. UPDATE ai_goal_suggestions SET status = 'accepted', accepted_goal_id = new_goal_id
   d. IF linked budget: UPDATE budgets SET linked_goal_id = new_goal_id, amount = target_amount
5. New goal card appears at top of Goals list
```

### Spending Habit Month Close (runs 00:01 on 1st)
```
FOR each active spending_habit goal:
1. SELECT sum(transactions) for closed month by category
2. INSERT goal_period_records (was_successful, variance, streak_at_close)
3. UPDATE goal_spending_habit (current_streak, longest_streak, totals)
4. IF was_successful: queue celebration push notification
5. IF current_streak = 0 AND total_periods_tracked > 2: queue re-engagement notification
6. IF auto_renew = true: goal remains active (no state change needed, month resets automatically)
```

### 30-Transaction Milestone
```
On every transaction review event:
1. UPDATE user_ai_activation SET reviewed_transaction_count = reviewed_transaction_count + 1
2. IF reviewed_transaction_count >= 30 AND milestone_reached = false:
   a. UPDATE user_ai_activation SET milestone_reached = true, activated_at = NOW()
   b. Run ai_goal_suggestion generation job for this user
   c. Queue "Totem AI activated" celebration
   d. UPDATE ai_insights_active = true (Dashboard AI Insights also activate)
```
