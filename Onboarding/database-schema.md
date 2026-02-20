# Totem — Database Schema
## Onboarding Data Architecture

> **Cross-reference:** The Onboarding flow primarily writes to the `users` table defined in `Dashboard/database-schema.md` and triggers the initial population of `accounts`, `transactions`, `ai_categorisation_queue`, and `transaction_reviews` (see their respective schemas). This file documents only the **additional tables** and **column additions** specific to onboarding.
>
> **Tables from Dashboard schema written during onboarding:**
> - `users` — created on sign-up; onboarding_step updated throughout
> - `accounts` — populated from Basiq connection
> - `transactions` — populated from Basiq 12-month pull
> - `ai_categorisation_queue` — populated as transactions are imported
> - `categorisation_rules` — any pre-existing rules applied
>
> **Tables from Accounts schema written during onboarding:**
> - `transaction_reviews` — created by categorisation worker
> - `account_connection_log` — first entry on successful bank connection
>
> **Database:** PostgreSQL (primary).
> **Conventions:** UUID PKs, TIMESTAMPTZ, INTEGER cents, soft deletes via `deleted_at`.

---

## Table of Contents
1. [Onboarding Sessions](#onboarding-sessions)
2. [Bank Connection Attempts](#bank-connection-attempts)
3. [Users Table — Onboarding Columns](#users-table--onboarding-columns)
4. [User Settings — Onboarding Keys](#user-settings--onboarding-keys)
5. [Feature → Table Dependency Map](#feature--table-dependency-map)
6. [Indexes](#indexes)
7. [Data Flow Notes](#data-flow-notes)

---

## Onboarding Sessions

### `onboarding_sessions`
Tracks each onboarding attempt (including abandoned ones). Used for analytics (drop-off by screen), debugging, and resuming partial onboarding if the user closes the app mid-flow.

```sql
CREATE TABLE onboarding_sessions (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID REFERENCES users(id) ON DELETE CASCADE,
        -- NULL for anonymous sessions (before account creation)

    -- Device / tracking
    device_id             VARCHAR(255),   -- app-generated device UUID (persisted locally)
    platform              VARCHAR(20),    -- 'ios' | 'android' | 'web'
    app_version           VARCHAR(20),

    -- Flow state
    current_step          VARCHAR(50) NOT NULL DEFAULT 'welcome',
        -- 'welcome' | 'account_creation' | 'bank_select' | 'pay_cycle'
        -- | 'bank_auth' | 'processing' | 'reveal' | 'review_transactions' | 'complete'

    selected_bank         VARCHAR(50),    -- 'anz' | 'cba' | 'westpac' | 'nab' | 'macquarie' | 'other'
    pay_cycle_selected    VARCHAR(20),    -- self-reported pay cycle from Screen 4
    explore_mode          BOOLEAN DEFAULT FALSE,  -- true if user chose "Explore with sample data"

    -- Bank connection result (populated after Screen 5)
    basiq_connection_id   VARCHAR(255),   -- Basiq connection ID on success
    bank_auth_attempted   BOOLEAN DEFAULT FALSE,
    bank_auth_succeeded   BOOLEAN DEFAULT FALSE,
    bank_auth_error       VARCHAR(100),   -- error code if failed

    -- Processing result (populated after Screen 6)
    transactions_imported INTEGER,        -- how many transactions were pulled
    processing_started_at TIMESTAMPTZ,
    processing_ended_at   TIMESTAMPTZ,
    processing_timed_out  BOOLEAN DEFAULT FALSE,

    -- Completion
    completed_at          TIMESTAMPTZ,    -- NULL if not yet complete
    abandoned_at          TIMESTAMPTZ,   -- NULL if not abandoned
    completed_cta         VARCHAR(30),    -- 'start_reviewing' | 'skip_to_app' | 'explore'

    started_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Usage:**
- Drop-off analytics: `WHERE completed_at IS NULL GROUP BY current_step`
- Resume onboarding: On app open, if `users.onboarding_complete = false`, fetch latest session to restore step
- A/B testing: onboarding_sessions can be tagged with experiment IDs via `user_settings`

---

## Bank Connection Attempts

### `bank_connection_attempts`
Fine-grained log of Basiq OAuth attempts during onboarding. Separate from `account_connection_log` (which tracks ongoing sync events post-onboarding).

```sql
CREATE TABLE bank_connection_attempts (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    onboarding_session_id UUID NOT NULL REFERENCES onboarding_sessions(id) ON DELETE CASCADE,
    user_id               UUID REFERENCES users(id) ON DELETE SET NULL,

    bank                  VARCHAR(50) NOT NULL,  -- 'anz' | 'cba' | etc.
    basiq_connector_id    VARCHAR(100),           -- Basiq's connector identifier

    -- Attempt outcome
    status                VARCHAR(20) NOT NULL DEFAULT 'initiated',
        -- 'initiated'   — WebView opened, user in auth flow
        -- 'succeeded'   — Basiq callback received, connection_id obtained
        -- 'failed'      — Basiq error or bank auth failed
        -- 'cancelled'   — User closed WebView without completing
        -- 'timeout'     — No callback received within 120s

    basiq_connection_id   VARCHAR(255),           -- populated on success
    error_code            VARCHAR(50),
    error_message         TEXT,

    initiated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at           TIMESTAMPTZ
);
```

**Usage:**
- Alert if success rate drops below threshold (bank outage detection)
- User support: reconstruct exactly what happened if a user reports connection issues

---

## Users Table — Onboarding Columns

The `users` table is defined in `Dashboard/database-schema.md`. The following columns are relevant to onboarding and are documented here for clarity:

```sql
-- Already defined in Dashboard/database-schema.md, included here for reference:
onboarding_complete   BOOLEAN NOT NULL DEFAULT FALSE,
onboarding_step       VARCHAR(50),   -- current step if incomplete (mirrors onboarding_sessions.current_step)

-- Pay cycle (set during Screen 4, refined post-bank-connect)
pay_cycle_type        VARCHAR(20),   -- 'weekly' | 'fortnightly' | 'monthly' | 'irregular'
pay_cycle_day_of_week SMALLINT,      -- 0=Mon..6=Sun
pay_cycle_day_of_month SMALLINT,     -- 1–31 for monthly

-- Income (NOT asked during onboarding — derived from bank feed)
income_is_irregular   BOOLEAN,
income_source_name    VARCHAR(255),
expected_net_income   INTEGER,       -- cents, detected from recurring income pattern

-- CSV import (not used in onboarding — Basiq replaces it for AU users)
csv_import_earliest_date DATE,
csv_import_latest_date   DATE,
```

### Onboarding Step State Machine
```sql
-- Valid values for users.onboarding_step:
-- 'welcome'              → default on account creation
-- 'bank_select'          → after account created, before bank chosen
-- 'pay_cycle'            → after bank selected, before pay cycle set
-- 'bank_auth'            → after pay cycle set, before Basiq OAuth complete
-- 'processing'           → Basiq OAuth succeeded, data pull in progress
-- 'reveal'               → processing complete, user viewing reveal screen
-- 'review_transactions'  → user on Screen 8
-- 'complete'             → onboarding done, onboarding_complete = true
-- 'skipped_bank'         → user chose "I'll connect later"
-- 'explore_mode'         → user chose "Explore with sample data" (no account)
```

---

## User Settings — Onboarding Keys

Stored in `user_settings` table (key-value, defined in Dashboard schema). Keys used during and after onboarding:

| Key | Type | Default | Purpose |
|---|---|---|---|
| `onboarding_explore_mode` | boolean | false | User entered via "Explore with sample data" |
| `onboarding_goals_nudge_shown` | boolean | false | Whether the Goals tab first-visit nudge has been shown |
| `onboarding_goals_nudge_dismissed_at` | timestamptz | null | When user dismissed the Goals nudge |
| `onboarding_insights_nudge_shown` | boolean | false | Whether the Insights tab nudge has been shown |
| `onboarding_bank_skipped` | boolean | false | User skipped bank connection in onboarding |
| `pay_cycle_auto_detected` | boolean | false | Whether pay cycle was auto-detected (vs user-reported) |
| `pay_cycle_detection_confidence` | string | null | 'high' \| 'medium' \| 'low' |
| `pay_cycle_user_confirmed` | boolean | false | User has confirmed the auto-detected pay cycle |

---

## Feature → Table Dependency Map

| Onboarding Screen | Writes To | Reads From |
|---|---|---|
| Screen 1: Welcome | — | — |
| Screen 2: Sign Up | `users`, `onboarding_sessions` | — |
| Screen 3: Bank Select | `onboarding_sessions` | — |
| Screen 4: Pay Cycle | `users` (pay_cycle_*), `onboarding_sessions` | — |
| Screen 5: Bank Auth | `bank_connection_attempts`, Basiq API | — |
| Screen 6: Processing | `accounts`, `transactions`, `ai_categorisation_queue`, `account_connection_log`, `onboarding_sessions` | Basiq API |
| Screen 7: The Reveal | — | `computed_metrics_cache`, `transactions`, `ai_insights`, `lifestyle_creep_snapshots` |
| Screen 8: First Task | `users` (onboarding_complete) | `transaction_reviews`, `transactions`, `categories` |
| Progressive nudges | `user_settings` | `user_ai_activation`, `user_settings` |

---

## Indexes

```sql
-- Onboarding sessions: resume on app open (most recent incomplete session)
CREATE INDEX idx_ob_sessions_user_step
    ON onboarding_sessions(user_id, current_step, last_seen_at DESC)
    WHERE completed_at IS NULL AND abandoned_at IS NULL;

-- Onboarding sessions: analytics drop-off queries
CREATE INDEX idx_ob_sessions_step_platform
    ON onboarding_sessions(current_step, platform, started_at DESC);

-- Bank connection attempts: support queries
CREATE INDEX idx_bank_conn_attempts_session
    ON bank_connection_attempts(onboarding_session_id);

CREATE INDEX idx_bank_conn_attempts_status
    ON bank_connection_attempts(bank, status, initiated_at DESC);
```

---

## Data Flow Notes

### App Launch → Resume Onboarding
```
On app open (authenticated user):
  IF users.onboarding_complete = false:
    Fetch latest onboarding_sessions WHERE user_id = $1 AND completed_at IS NULL
    IF session exists:
      Restore to session.current_step screen
    ELSE:
      Start fresh from users.onboarding_step
```

### Anonymous Explore Mode
```
"Explore with sample data" tapped:
  1. No users record created
  2. onboarding_sessions row created with user_id = NULL, explore_mode = true
  3. device_id stored locally
  4. Sample data (Alex's dataset) loaded into app state (not from DB)
  5. When user later taps "Connect your bank":
     → Show account creation screen (Screen 2) first
     → After account creation: link device_id session to new user_id
     → Continue from Screen 3 (bank select)
```

### Basiq Processing Pipeline
```
Triggered after bank_connection_attempts.status → 'succeeded':

1. INSERT account_connection_log (event_type = 'initial_connect')
2. Fetch account list from Basiq → INSERT accounts rows
3. Begin transaction stream from Basiq (CDR: up to 12 months):
   FOR EACH transaction chunk from Basiq:
     INSERT INTO transactions (source = 'bank_feed')
     INSERT INTO ai_categorisation_queue (status = 'queued')
     EMIT SSE event to frontend: { type: 'import_progress', count: N }
4. When all transactions imported:
   EMIT SSE: { type: 'import_complete', total: N }
   UPDATE onboarding_sessions SET transactions_imported = N
5. Categorisation worker (was already running concurrently):
   Processes ai_categorisation_queue in batches
   EMIT SSE events for categorisation progress (optional — Screen 6 shows step completion)
6. After categorisation complete:
   Run pay cycle detection → update users.pay_cycle_*
   Run recurring bill detection → INSERT recurring_bills
   Batch compute all historical metric_snapshots
   Compute spend_trends
   Compute forecast_cache for next 3 months
   Populate all computed_metrics_cache keys
   Generate ai_insights (initial set)
   EMIT SSE: { type: 'processing_complete' }
7. Frontend receives 'processing_complete' → advance to Screen 7
```

### Onboarding Completion
```
When user taps "Start reviewing →" or "Skip for now" on Screen 8:
  UPDATE users SET onboarding_complete = true, onboarding_step = 'complete'
  UPDATE onboarding_sessions SET completed_at = NOW(), completed_cta = 'start_reviewing' / 'skip_to_app'
  IF 'start_reviewing': navigate to Accounts screen, To Review tab
  IF 'skip_to_app':     navigate to Dashboard (home screen)
```

### Pay Cycle Refinement (post-onboarding)
```
After processing_complete, pay cycle detection runs:
  Detected cycle stored in computed_metrics_cache temporarily
  IF detected cycle ≠ user-reported cycle:
    SET user_settings['pay_cycle_detection_confidence'] = detected_confidence
    SET user_settings['pay_cycle_auto_detected'] = true
    On next Dashboard open: show "We detected you're paid fortnightly on Thursdays. Is that right?"
    IF user confirms → UPDATE users.pay_cycle_* to detected values
    IF user denies   → keep user-reported values, SET user_settings['pay_cycle_user_confirmed'] = true
```
