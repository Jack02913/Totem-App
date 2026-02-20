# Totem — Database Schema
## Accounts Screen Data Architecture

> **Cross-reference:** The Accounts screen primarily reads from tables defined in `Dashboard/database-schema.md`. This file documents only the **additional tables** introduced by the Accounts screen's specific features.
>
> **Tables from Dashboard schema used by Accounts:**
> - `users` — user identity and settings
> - `accounts` — linked bank accounts (balances, types, sync status)
> - `transactions` — all transaction records
> - `categories` — category definitions
> - `categorisation_rules` — auto-categorisation rules (defined in Dashboard schema, managed here)
>
> **Database:** PostgreSQL (primary). Redis for live feed status caching.
> **Conventions:** UUID PKs, TIMESTAMPTZ, INTEGER cents, soft deletes via `deleted_at`.

---

## Table of Contents
1. [Account Connection Log](#account-connection-log)
2. [Transaction Reviews](#transaction-reviews)
3. [AI Categorisation Queue](#ai-categorisation-queue)
4. [Cross-Reference: Categorisation Rules](#cross-reference-categorisation-rules)
5. [Feature → Table Dependency Map](#feature--table-dependency-map)
6. [Indexes](#indexes)
7. [Data Flow Notes](#data-flow-notes)

---

## Account Connection Log

### `account_connection_log`
Records every sync attempt with the bank feed connector (Basiq / NZ). Used for debugging connection issues, displaying last sync time, and the live feed indicator.

```sql
CREATE TABLE account_connection_log (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    account_id            UUID REFERENCES accounts(id) ON DELETE SET NULL,
        -- NULL if the sync was at the connection level (not per-account)

    -- Connector details
    connector             VARCHAR(30) NOT NULL,  -- 'basiq_au' | 'akahu_nz' | 'manual'
    external_event_id     VARCHAR(255),          -- Basiq job ID or webhook event ID

    -- Sync outcome
    event_type            VARCHAR(30) NOT NULL,
        -- 'initial_connect' | 'webhook_sync' | 'scheduled_refresh' | 'manual_refresh'
        -- | 'balance_reconciliation' | 'disconnect'

    status                VARCHAR(20) NOT NULL,
        -- 'success' | 'partial' | 'failed' | 'pending'

    transactions_added    SMALLINT DEFAULT 0,    -- new transactions inserted this sync
    transactions_updated  SMALLINT DEFAULT 0,    -- existing transactions updated
    balance_updated       BOOLEAN DEFAULT FALSE, -- was account.balance refreshed?

    -- Error details (if status = 'failed')
    error_code            VARCHAR(50),
    error_message         TEXT,

    started_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at          TIMESTAMPTZ,

    -- Derived: completed_at - started_at (computed at query time or stored)
    duration_ms           INTEGER
);
```

**Usage:**
- `MAX(completed_at) WHERE status = 'success' AND account_id = ?` → "Last synced" timestamp for live feed indicator
- Failed syncs with the same error_code repeated 3× in 24h → trigger user notification: "We're having trouble connecting to [Bank]"

---

## Transaction Reviews

### `transaction_reviews`
Tracks the review state of each transaction in the To Review queue. One row per transaction that has entered the review queue. Transactions that were auto-categorised by a rule (category_assigned_by = 'rule') skip the queue entirely and never get a row here.

```sql
CREATE TABLE transaction_reviews (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    transaction_id        UUID NOT NULL REFERENCES transactions(id) ON DELETE CASCADE,

    -- Queue state
    review_status         VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- 'pending'   — in queue, not yet reviewed
        -- 'approved'  — user tapped ✓, accepted AI suggestion
        -- 'corrected' — user tapped ✗ and selected a different category
        -- 'skipped'   — user dismissed without categorising (future feature)
        -- 'auto_approved' — auto-approved by rule match after correction learning

    -- AI suggestion at time of queue entry (snapshot — category may change on correction)
    ai_suggested_category_id   UUID REFERENCES categories(id),
    ai_confidence_score        NUMERIC(3,2),   -- 0.00–1.00
    ai_confidence_label        VARCHAR(10),    -- 'high' | 'medium' | 'low'

    -- Correction details (populated if review_status = 'corrected')
    corrected_category_id      UUID REFERENCES categories(id),
    correction_source          VARCHAR(20),    -- 'user_picker' | 'rule_applied'

    -- Timestamps
    queued_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    reviewed_at           TIMESTAMPTZ,        -- when user tapped ✓ or ✗

    UNIQUE(transaction_id)   -- one review record per transaction
);
```

**Key queries:**

Count reviewed transactions (for progress bar):
```sql
SELECT COUNT(*) FROM transaction_reviews
WHERE user_id = $1
  AND review_status IN ('approved', 'corrected', 'auto_approved');
```

Fetch pending queue (ordered for display):
```sql
SELECT tr.*, t.merchant_name, t.amount, t.date, c.name AS suggested_category_name, c.icon
FROM transaction_reviews tr
JOIN transactions t ON t.id = tr.transaction_id
JOIN categories c ON c.id = tr.ai_suggested_category_id
WHERE tr.user_id = $1 AND tr.review_status = 'pending'
ORDER BY
  tr.ai_confidence_label DESC,  -- high → medium → low
  t.amount DESC                  -- largest amounts first within tier
LIMIT 20;
```

---

## AI Categorisation Queue

### `ai_categorisation_queue`
A job queue table that tracks the async AI categorisation work for each new transaction. Separate from `transaction_reviews` — this is the backend processing state, not the user-facing review state.

```sql
CREATE TABLE ai_categorisation_queue (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    transaction_id        UUID NOT NULL REFERENCES transactions(id) ON DELETE CASCADE,

    -- Processing state
    status                VARCHAR(20) NOT NULL DEFAULT 'queued',
        -- 'queued'     — waiting to be picked up by categorisation worker
        -- 'processing' — currently being categorised
        -- 'complete'   — category assigned, transaction_review row created
        -- 'failed'     — categorisation failed (will retry up to 3×)
        -- 'skipped'    — skipped (e.g. transaction is a transfer, income, or rule-matched)

    -- Attempt tracking
    attempt_count         SMALLINT NOT NULL DEFAULT 0,
    last_attempted_at     TIMESTAMPTZ,
    last_error            TEXT,

    -- Result (populated on complete)
    assigned_category_id  UUID REFERENCES categories(id),
    confidence_score      NUMERIC(3,2),
    model_version         VARCHAR(50),        -- which AI model version was used

    -- Skip reason (populated on skipped)
    skip_reason           VARCHAR(50),
        -- 'is_transfer' | 'is_income' | 'rule_matched' | 'duplicate'

    queued_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at          TIMESTAMPTZ,

    UNIQUE(transaction_id)
);
```

**Worker logic (simplified):**
```
LOOP:
  Pick up to 50 rows WHERE status = 'queued' OR
    (status = 'failed' AND attempt_count < 3 AND last_attempted_at < NOW() - INTERVAL '5 minutes')
  ORDER BY queued_at ASC

  FOR each row:
    SET status = 'processing', attempt_count = attempt_count + 1

    IF transaction matches any active categorisation_rule:
      → apply rule category
      SET status = 'skipped', skip_reason = 'rule_matched'
      UPDATE transactions SET category_id, category_assigned_by = 'rule', category_confidence = 1.00
      // No transaction_review row created — rule-matched txns skip the To Review queue
      CONTINUE

    IF transaction.is_transfer OR transaction.is_income:
      SET status = 'skipped', skip_reason = (is_transfer OR is_income)
      CONTINUE

    → Call AI categorisation model with: merchant_name, description, amount, user_id
    → Receive: category_id, confidence_score

    UPDATE transactions SET category_id, category_assigned_by = 'ai', category_confidence
    INSERT INTO transaction_reviews (transaction_id, ai_suggested_category_id, ai_confidence_score, ai_confidence_label, ...)
    SET status = 'complete'
```

---

## Cross-Reference: Categorisation Rules

The `categorisation_rules` table is **defined in `Dashboard/database-schema.md`** and is the source of truth for its schema. The Accounts screen (Rules tab) reads from and writes to this table.

Key operations from the Accounts screen:
- **Rules tab display:** `SELECT * FROM categorisation_rules WHERE user_id = $1 AND is_active IS NOT NULL ORDER BY created_by = 'user' DESC, created_at DESC`
- **Toggle rule:** `UPDATE categorisation_rules SET is_active = NOT is_active WHERE id = $1`
- **Delete rule:** `DELETE FROM categorisation_rules WHERE id = $1`
- **User creates rule:** `INSERT INTO categorisation_rules (user_id, match_type, match_value, category_id, created_by = 'user')`
- **AI suggests rule** (from correction learning): `INSERT INTO categorisation_rules (..., created_by = 'ai_suggestion')`

### Correction-to-Rule Learning
```sql
-- Check if merchant has been corrected to same category 2+ times
SELECT corrected_category_id, COUNT(*) as correction_count
FROM transaction_reviews tr
JOIN transactions t ON t.id = tr.transaction_id
WHERE tr.user_id = $1
  AND tr.review_status = 'corrected'
  AND t.merchant_name_normalised = $2  -- the merchant being corrected
GROUP BY corrected_category_id
HAVING COUNT(*) >= 2;

-- If found: auto-insert an AI rule
INSERT INTO categorisation_rules (
    user_id, match_type, match_value,
    category_id, created_by, is_active
) VALUES (
    $1, 'merchant_name', $2_normalised,
    $3_category_id, 'ai_suggestion', true
)
ON CONFLICT DO NOTHING;  -- prevent duplicate rules for same merchant
```

---

## Feature → Table Dependency Map

| Accounts Feature | Primary Tables | Secondary / Supporting |
|---|---|---|
| **Account Linking / Live Feed** | `accounts`, `account_connection_log` | Basiq API |
| **Net Position hero** | `accounts` | — |
| **Account Detail sheet** | `accounts`, `transactions`, `categories` | — |
| **To Review queue** | `transaction_reviews`, `transactions` | `categories`, `ai_categorisation_queue` |
| **Approve transaction** | `transaction_reviews` | `user_ai_activation` (milestone check) |
| **Correct category** | `transaction_reviews`, `transactions` | `categorisation_rules` (learning) |
| **30-transaction gate** | `transaction_reviews` | `user_ai_activation` |
| **Rules tab** | `categorisation_rules`, `categories` | — |
| **Toggle rule** | `categorisation_rules` | — |
| **AI rule suggestion** | `transaction_reviews`, `categorisation_rules` | `transactions` |
| **AI categorisation (backend)** | `ai_categorisation_queue`, `transactions` | `categorisation_rules` |

---

## Indexes

```sql
-- Account connection log: live feed indicator query
CREATE INDEX idx_acct_conn_log_user_status
    ON account_connection_log(user_id, status, completed_at DESC);

CREATE INDEX idx_acct_conn_log_account
    ON account_connection_log(account_id, completed_at DESC);

-- Transaction reviews: queue fetch and count
CREATE INDEX idx_txn_reviews_user_status
    ON transaction_reviews(user_id, review_status);

CREATE INDEX idx_txn_reviews_user_pending
    ON transaction_reviews(user_id, ai_confidence_label, queued_at)
    WHERE review_status = 'pending';

CREATE INDEX idx_txn_reviews_transaction
    ON transaction_reviews(transaction_id);

-- AI categorisation queue: worker pickup
CREATE INDEX idx_ai_cat_queue_status
    ON ai_categorisation_queue(status, queued_at)
    WHERE status IN ('queued', 'failed');

CREATE INDEX idx_ai_cat_queue_transaction
    ON ai_categorisation_queue(transaction_id);
```

---

## Data Flow Notes

### New Transaction → Queue → Review
```
1. Basiq webhook fires → transaction inserted into transactions table
2. ai_categorisation_queue row inserted (status = 'queued')
3. Categorisation worker picks up job:
   a. Check categorisation_rules → if rule matches: apply, set skip_reason = 'rule_matched', skip queue
   b. If income/transfer: set skip_reason, skip queue
   c. Otherwise: call AI model → insert transaction_reviews row (status = 'pending')
4. User opens To Review tab → sees pending reviews
5. User taps ✓ (approve) or ✗ (correct):
   - Update transaction_reviews.review_status
   - If correct: update transactions.category_id
   - Increment review count display
   - Check 30-transaction milestone
6. If correction: check if 2+ corrections for same merchant → insert ai categorisation rule
```

### Live Feed Indicator Refresh
```
While Accounts screen is active:
  EVERY 60 seconds:
    Fetch MAX(completed_at) WHERE status = 'success' AND account_id IN (user's accounts)
    → Update dot colour (green/amber/red) based on recency
    → Update "Last updated X min ago" tooltip text
```

### 30-Transaction Milestone Fire
```
On every approve or correct action:
  reviewed_count = SELECT COUNT(*) FROM transaction_reviews
                   WHERE user_id = $1 AND review_status IN ('approved', 'corrected', 'auto_approved')

  IF reviewed_count = 30 AND NOT EXISTS (
      SELECT 1 FROM user_ai_activation WHERE user_id = $1 AND milestone_type = 'thirty_transactions'
  ):
    INSERT INTO user_ai_activation (user_id, milestone_type, reached_at)
    VALUES ($1, 'thirty_transactions', NOW())

    → Trigger: unlock AI goal suggestions
    → Trigger: switch categorisation model to user-personalised variant
    → Fire: milestone celebration animation in UI
```
