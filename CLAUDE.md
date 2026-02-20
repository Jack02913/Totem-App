# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Totem** is a personal finance app targeting 18–30 year olds in Australia and New Zealand. The core value proposition is eliminating the "rear-view mirror" problem of personal finance — using direct bank feeds (Basiq for AU, Akahu equivalent for NZ) and AI auto-categorisation to give users up-to-date, proactive financial insights rather than passive historical reporting.

Key differentiators:
- Live bank feeds via Basiq (AU) / NZ open banking — no manual CSV exports required
- Per-user AI categorisation that learns over time
- Proactive push notifications driven by ML (e.g. identify high-risk takeout days, notify the evening before)
- Novel metrics: Lifestyle Creep, Safety Net runway, Savings Velocity
- Available on iOS, Android (React Native), and Web (React)

## Repository Structure

```
totem-mobile-mvp/
├── Dashboard/              ← Screen 1 (complete)
│   ├── feature-list.md     ← All features with formulas, algorithms, data dependencies
│   ├── design-spec.md      ← Visual system, component specs, layout proportions
│   ├── database-schema.md  ← PostgreSQL schema, indexes, data flow
│   └── dashboard-mockup.html ← Standalone screen prototype
├── Plan/                   ← Screen 2 (complete) — Goals + Budgets sub-tabs
│   ├── feature-list.md
│   ├── design-spec.md
│   ├── database-schema.md  ← Supersedes savings_goals from Dashboard schema
│   └── plan-mockup.html
├── Calendar/               ← Screen 3 (complete) — Monthly expense calendar
│   ├── feature-list.md
│   ├── design-spec.md
│   └── database-schema.md  ← Primarily a read layer; adds calendar_day_cache table
├── Accounts/               ← Screen 4 (complete)
│   ├── feature-list.md     ← Account linking, Net Position, To Review queue, rule engine
│   ├── design-spec.md      ← 3-tab layout, account detail sheet, category picker, rule rows
│   └── database-schema.md  ← transaction_reviews, ai_categorisation_queue, account_connection_log
├── Insights/               ← Screen 5 (complete)
│   ├── feature-list.md     ← Spend/Trends/Income/Forecast tab features and formulas
│   ├── design-spec.md      ← Dual bar chart, forecast hero, risk flags, period chips
│   └── database-schema.md  ← metric_snapshots, forecast_cache, spend_trends tables
└── Onboarding/             ← Pre-app flow (complete — prototype in app-mockup.html)
    ├── feature-list.md     ← 8-screen flow, state machine, progressive nudges, Basiq pipeline
    ├── design-spec.md      ← Overlay architecture, ob-screen transitions, all component specs
    └── database-schema.md  ← onboarding_sessions, bank_connection_attempts, user settings keys
```

**Each screen gets its own folder** with the same four files. All five app screens and the onboarding flow now have full documentation.

**`app-mockup.html`** (root of repo) is the **unified prototype** — all screens in one file, navigable via the bottom nav. This is the primary UX assessment tool. Individual screen mockups inside each folder are standalone references only.

### Unified Mockup Architecture (`app-mockup.html`)
- Single `<style>` block with all CSS custom properties — changing a theme variable propagates to every screen instantly
- Three built-in themes: Midnight Blue (default, `:root`), OLED Black (`.t-oled`), Light (`.t-light`)
- Each screen is an `.app-screen` div (`position: absolute`, opacity/transform transitions) — screens slide in/out on nav tap
- Persistent elements outside `.app-screen`: bottom nav, FAB, AI chat overlay, status bar
- When building new screens: add a new `.app-screen` div, add a nav item, add a sidebar entry, add a notes block — no CSS changes needed unless the screen introduces new component types
- Dashboard tier-switching (← → keys, swipe) only activates when `currentScreen === 'home'`
- Bottom sheets (`.sheet-backdrop` + `.bottom-sheet`) live outside `.app-screen` inside `.phone-screen`; each sheet has its own backdrop `id` and close function
- `window._febGridHTML` caches the February calendar grid HTML on init for month-nav restore
- All `acct-` prefixed IDs/functions belong to the Accounts screen; all `ins-` prefixed belong to Insights; all `ob-` prefixed belong to the Onboarding overlay

## Documentation Conventions

- **Amounts** are always stored as `INTEGER` cents (e.g. $18.99 → `1899`). Formatting to display currency is a frontend concern.
- **Timestamps** are `TIMESTAMPTZ` (timezone-aware). User timezone is stored on the `users` table.
- **UUIDs** for all primary keys (`gen_random_uuid()`).
- **Soft deletes** via `deleted_at` on user-facing tables; hard deletes elsewhere.
- All feature documentation in `feature-list.md` includes: formula, edge cases, data dependencies, and calculation run schedule (webhook-triggered vs scheduled).

## Established Architecture Decisions

### Database
- PostgreSQL primary, Redis for computed metric caching
- `computed_metrics_cache` table provides persistent fallback when Redis is cold
- All dashboard metrics recompute on Basiq webhook (new transaction) and on app open
- Monthly lifestyle creep snapshots run at 02:00 AEST on the 1st of each month

### Dashboard — 3-Tier System
The dashboard uses a horizontally swipeable 3-tier system on mobile:
- **Tier 1 (Overview):** Safe to Spend, Month Projection, AI Insight, Upcoming Bills — daily decision screen, must fit on one viewport without scrolling
- **Tier 2 (Financial Health):** Budget Burn Rate, Subscription Load, Safety Net, Savings Velocity — unlocks after first full pay period of data
- **Tier 3 (Trends):** Lifestyle Creep (builds over 3 months), Category % of Income, Savings Trend

On desktop (≥1024px), all three tiers display simultaneously in a 2-column layout (Tier 1 + Tier 2 side-by-side, Tier 3 full-width below). The swipe metaphor is desktop-only replaced with visible columns.

### Key Metric Definitions
- **Safe to Spend** = Net pay period income − already spent (discretionary) − committed bills before payday − scheduled savings transfers. Scoped to the current pay period, not calendar month.
- **Safety Net** = Liquid savings ÷ 3-month rolling average monthly expenses (excludes super and investments).
- **Lifestyle Creep** = spend_growth_3mo_vs_12mo% − income_growth_3mo_vs_12mo%. Requires 3 months minimum data; shows "building baseline" state until then. Users can skip this by uploading 12 months of CSV history on onboarding.
- **AI Insight** scoring: `(severity × 0.35) + (urgency × 0.30) + (dollar_impact_score × 0.25) + (novelty × 0.10)`. One insight shown at a time; highest score wins.

### Pay Cycle Handling
Pay cycle is auto-detected from bank feed patterns (3+ consistent deposits from same source, stddev < 4 days). Users with irregular income get a fallback mode: 30-day rolling window replaces pay-period framing, and Safe to Spend uses account balance minus committed 30-day bills.

### Recurring Bill Detection
Transactions grouped by `merchant_name_normalised` (strip PTY LTD, trailing numbers, etc.). Pattern flagged after 2 occurrences; high confidence at 3+. Requires user approval before appearing in Upcoming Bills or Safe to Spend calculations. Confidence levels: high (stddev < 3 days, amount variance < 5%), medium (stddev < 5 days, variance < 15%), low (otherwise — not shown without user enabling).

### Plan Screen — Goals + Budgets (Tab 2)
Three goal types with distinct visual identities (left-border colour coding):
- **Save Up** (purple/brand): accumulate toward a target amount; optional auto-transfer; status Ahead/On Track/At Risk/Behind/In Progress
- **Spending Habit** (teal): Totem's unique goal type — reduce category spend by X% from a 3-month baseline (immutable at creation); auto-renews monthly; streak tracking (1→muted, 2-3→amber, 4+→brand)
- **Debt Paydown** (green): tracks credit card/loan balance reduction; avg monthly payment from last 3 months

**30-Transaction Milestone**: gamified AI activation gate — progress bar shown in Goals tab until 30 reviewed transactions are reached, at which point AI goal suggestions and insights activate.

**Budget-Goal Sync**: `budgets.linked_goal_id` FK added via migration to `Dashboard/database-schema.md`'s budgets table. When linked, `budget.amount` stays synced with `goal_spending_habit.target_amount`.

**`savings_goals` table from Dashboard schema is superseded** by the comprehensive goals schema in `Plan/database-schema.md` (8 tables: `goals`, `goal_save_up`, `goal_spending_habit`, `goal_debt_paydown`, `goal_period_records`, `ai_goal_suggestions`, `goal_contributions`, `user_ai_activation`).

### Calendar Screen — Monthly Expense Calendar (Screen 3)
7-column (Sun–Sat) monthly grid. Key design decisions:
- **Safety tinting**: day cell backgrounds (green/amber/red) based on daily spend vs daily discretionary budget — Totem's primary visual differentiator from competitors
- **Category dots**: up to 3 coloured dots per cell (red=eating out, green=groceries, teal=transport, amber=entertainment) — spending fingerprint at a glance
- **Payday cells**: brand-tinted background, "Pay" micro-label, net amount in brand purple
- **Bill flags**: amber/red dot in top-right corner of future cells with confirmed recurring bills due
- **Day detail**: tap any cell → bottom sheet with full transaction list (past) or upcoming bills (future)
- **Month navigation**: ‹ › switches between months; `calendar_day_cache` table powers fast rendering
- **Daily budget denominator**: `sum(budgets.amount WHERE category.is_discretionary) / days_in_month`
- Mock data: Alex, Feb 2026. Today = Thu 19. Payday Feb 8 (past) + Feb 22 (upcoming). Bills: Netflix Feb 20, Rent Feb 21, Spotify Feb 22.

### Accounts Screen — Screen 4 (complete in app-mockup.html)
4-tab layout: **Accounts | To Review | Rules | Subs**

**Accounts tab:**
- Net Position hero card = Assets − Debt (Alex: +$13,454 = $14,297 assets − $843 debt)
- 3 linked accounts: ANZ Everyday ($1,847), ANZ NetSaver ($12,450 savings at 3.75%), ANZ Credit Visa (−$843)
- Pulsing green dot = live Basiq feed indicator
- Tap any account row → detail bottom sheet with last 5 transactions

**To Review tab:**
- The 30-transaction AI activation gate (Alex is at 24/30 = 80%)
- Each item shows: merchant, date, amount, AI-suggested category pill, confidence badge (High/Medium)
- ✓ button: smooth CSS collapse animation (max-height transition), increments counter, updates progress bar
- ✗ button: opens category picker sheet (11 categories in 3-col grid); picking a category then auto-approves the transaction
- Empty state appears when all 6 are reviewed; milestone turns green at 30/30

**Rules tab:**
- 8 auto-categorisation rules: 4 user-created ("You" badge), 4 AI-detected ("✨ AI" badge)
- Each rule row is clickable (`data-merchant`, `data-cat`, `data-emoji` attributes) → opens `acct-rule-edit-sheet`
- Rule edit sheet: shows merchant context row + 11-category picker grid; save updates the row's icon, label, `data-cat`; AI badge downgrades to "You" on edit
- ON/OFF toggle and trash icon use `event.stopPropagation()` to prevent triggering row click
- Add Rule sheet (`acct-add-rule-sheet`) uses separate `add-rule-cat-grid` ID and `pickAddRuleCat()` to avoid ID collision with edit sheet

**Subs tab:**
- 11 active subscription rows in a `.list-card` inside `.tab-pane#acct-pane-subs`
- Each row: `.list-icon` rounded-square icon (brand-tinted per service colour) + merchant name + category tag + monthly cost
- Active subscriptions: Netflix $18.99 · Spotify $12.99 · Apple TV+ $12.99 · YouTube Premium $22.99 · Disney+ $13.99 · iCloud+ $4.99 · Amazon Prime $9.99 · ChatGPT Plus $29.99 · F45 Training $65 · Adobe CC $89.99 · Headspace $12.99
- Header action shows "11 active" (static label, no onclick)
- Dashboard "Manage subscriptions" → `navigateToAcctSubs()` which calls `navigate('accounts')` then `switchAcctTab('subs')` after 300ms delay

**Bottom sheets:** `acct-acct-sheet` (account detail), `acct-cat-sheet` (To Review category picker), `acct-rule-edit-sheet` (rule edit — 11 categories in 3-col grid), `acct-add-rule-sheet` (add new rule)

**Key JS functions:** `switchAcctTab`, `updateAcctHeader`, `approveTransaction`, `openCatSheet`, `closeCatSheet`, `pickCategory`, `openAcctDetail`, `closeAcctSheet`, `navigateToAcctSubs`, `openRuleEdit`, `pickRuleCat`, `saveRuleEdit`, `closeRuleEdit`, `pickAddRuleCat`

**State:** `acctReviewCount` (starts at 24), `acctTotalGoal` (30), `acctPendingTxn` (txn ID during category picker flow), `currentRuleEl` (rule row element during edit flow)

### Insights Screen — Screen 5 (complete in app-mockup.html)
4-tab layout: **Spend | Trends | Income | Forecast**

**Spend tab:**
- Period chips: This Month / 3 Months / 12 Months — fully interactive (`switchInsPeriod(period, chipEl)`), switches all KPI values, income label, and all 10 category bar widths/percentages/amounts
- KPI row: Total Spent $2,781 · Daily avg $146 · Saved 18% (This Month defaults)
- Category breakdown: 10 categories with horizontal bars scaled to % of $6,400 monthly income; red bar = Dining Out (over 5% threshold)
- Top 5 merchants with proportional fill bars
- AI insight card (Dining Out creep flagged)

**Trends tab:**
- **Lifestyle Creep** (full activated state): spend growth +11% vs income growth +5% = **+6% creep delta** (amber warning). Main driver: Dining Out. Breakdown rows show the formula components.
- **Safety Net**: 2.6 months runway (amber — below 3-month target), $12,450 liquid savings
- **Monthly Spend Trend**: 5-bar chart Oct–Feb; Dec spike in red (holiday season); Feb marked as partial
- **Savings Velocity**: Oct–Feb mini bar chart; Jan peak (27%), Feb current (18%)

**Income tab:**
- KPI row: Monthly $6,400 · YTD $9,640 · Growth +5%
- Dual bar chart: income (brand) vs spend (red) per month Oct–Feb. Feb income bar shown dashed/partial (one of two fortnightly pays landed)
- Pay history: last 4 fortnightly Acme Corp payments ($3,200–$3,240)
- Savings breakdown: ANZ NetSaver transfer ($500/$500) + Japan Trip goal ($280/$300)

**Forecast tab:**
- **Month-end projection hero**: +$2,241 projected Feb surplus (income $6,480 − projected spend $4,239)
- **90-day cash flow**: Mar/Apr/May paired income vs projected spend bars; avg projected surplus +$2,180/month
- **Goal ETAs**: Japan Trip → May 2026 (68%), Safety Net 3mo → Apr 2026 (86%), New Car Deposit → Dec 2026 (31%)
- **Upcoming known expenses**: Rent Feb 21, Car rego Mar 15 ($680), Japan flights deposit Apr 1 ($1,200)
- **AI flags**: 4 items — Dining Out risk (red), Car rego heads-up (amber), Japan opportunity (green), Safety Net milestone (brand)

**Key JS functions:** `switchInsTab(tab)`, `switchInsPeriod(period, chipEl)` — period chip handler; data object `insSpendData` holds This Month / 3 Months / 12 Months datasets (KPI values, income label, 10 category rows each with emoji, name, pct, bar width, colour, amount)

### Onboarding Flow (complete in app-mockup.html)
8-screen flow rendered as a full-screen overlay (`z-index: 35`, `top: 54px` to keep status bar visible) above all app screens and the bottom nav.

**3 phases + progressive in-app:**
- **Phase 1 — Hook**: Screen 1 (Welcome / "Explore with sample data" / "Get started") → Screen 2 (Account creation — Apple or email/password, minimal fields)
- **Phase 2 — Connect**: Screen 3 (Bank select — ANZ/CBA/Westpac/NAB/Macquarie/Other via Basiq) → Screen 4 (Pay cycle self-report; income NOT asked — auto-detected from feed) → Screen 5 (Bank auth — Basiq CDR OAuth, in-app WebView) → Screen 6 (Processing animation — Basiq import + AI categorisation + metrics pipeline, SSE-driven)
- **Phase 3 — Reveal + First Task**: Screen 7 (The Reveal — real Safe to Spend, top category, savings rate, AI flag) → Screen 8 (Transaction review preview — 30-transaction gate, 3 preview items)
- **Phase 4 — Progressive**: AI chat bubble nudges on first Goals tab visit and first Insights tab visit; not a dedicated screen

**Key design decisions:**
- "Explore with sample data" creates no user record; device_id stored locally; linked to user_id on later account creation
- Pay cycle auto-refined from bank feed after import; user shown confirmation card on first Dashboard open if detection differs
- Basiq CDR pulls up to 12 months of history automatically — no CSV import needed for AU major banks
- Screen 7 (The Reveal) is the highest-converting moment; both CTAs advance to Screen 8 (corrections happen through review process)

**Key JS functions:** `showOnboarding()`, `obTo(id)`, `obExplore()`, `obComplete()`, `obSelectBank(el)`, `obSelectCycle(el)`, `obStartConnect()`, `obRunProcessing()`

**Sidebar entry:** "Prototype Flows → Onboarding Flow" calls `showOnboarding()`. Onboarding notes shown in `notes-onboarding`.

**State machine** (`users.onboarding_step`): `null` → `account_created` → `bank_connect` → `pay_cycle` → `bank_auth` → `processing` → `reveal` → `review_transactions` → `complete`. Escape hatches: `skipped_bank`, `explore_mode`.

### Financial Archetype System — AI Personalisation
Totem uses a psychographic classification system based on the Klontz Money Script Inventory (KMSI-R) to adapt the AI chat assistant's tone and framing to each user's relationship with money.

**Trigger**: Post-first-value — fires after the user's *first* AI chat exchange (not on open). `sendChat()` flips `chatFirstMsg = false` on the first user message, then after the AI responds it calls `_offerQuiz()`. Input is locked (`quizState === 'running'` guard) during the quiz so typed messages cannot interrupt.

**4 Archetypes:**
| Archetype | Label | Icon | Colour | Core belief |
|---|---|---|---|---|
| The Detacher | Money Avoidance | 🌊 | `--amber` | Money is stressful or morally complicated |
| The Chaser | Money Worship | 🚀 | `--brand` | More money = more happiness |
| The Achiever | Money Status | 🏆 | `--teal` | Self-worth tied to net worth |
| The Guardian | Money Vigilance | 🛡️ | `--green` | Cautious, savings-focused, anxious |

**Quiz**: 6 questions (KMSI-R adapted for consumer UX), each with 4 answer chips (one per archetype). Rendered entirely in-chat — one question per AI bubble, chips below. Scoring: tally per archetype; most answers wins. Tiebreak priority: vigilance > status > worship > avoidance.

**Result card** shown in-chat after Q6: `.archetype-card` with colour-coded `border-left`, icon, name, label, description paragraph, and AI communication commitment line.

**Post-quiz UI:**
- **Chat header chip** (`.chat-archetype-chip#chat-archetype-chip`): hidden by default, gains `.visible` class on quiz completion — shows `{icon} {name}`, tapping opens profile sheet
- **Profile sheet** (`dash-profile-sheet`): archetype banner (`#profile-arch-banner`, hidden until quiz done) pre-populated by `_populateArchetypeProfile()`; "Retake personality quiz" → `retakeQuiz()` resets all state and restarts quiz

**AI communication adaptations (behaviour, not UI):**
- **Detacher**: Normalise money conversations, reduce shame, frame as self-care, celebrate small wins
- **Chaser**: Reality-check lifestyle creep, celebrate non-monetary wins, help reframe "enough"
- **Achiever**: Focus on personal benchmarks not comparisons, authentic goal-setting
- **Guardian**: Data-rich responses, validate caution, affirm that values-spending is OK

**Key JS state:** `chatFirstMsg` (bool — triggers quiz offer), `quizState` (idle | offered | running | done | skipped), `quizScores` ({avoidance, worship, status, vigilance}), `userArchetype` (string | null)

**Key JS functions:** `sendChat()`, `_offerQuiz()`, `beginQuiz()`, `skipQuiz()`, `_showQuizQ(idx)`, `answerQ(btn,arch,idx)`, `_calcArchetype()`, `_showResult()`, `_populateArchetypeProfile()`, `retakeQuiz()`

**Key data constants:** `archetypes` (obj keyed by archetype — name, label, icon, color, desc, aiNote), `quizQs` (array of 6 — each has `q` string and `opts` array of {t, a})

### Navigation Architecture (complete)
5-tab bottom nav: **Home** (Dashboard), **Plan** (Goals + Budgets), **Calendar**, **Accounts**, **Insights**. AI Chat is a persistent floating action button (✨ FAB) above the bottom nav on every screen — not a nav tab. This is the Monarch/Nova pattern: chat as an overlay layer.

The `navigate(id)` function handles:
- Screen transition (opacity/transform)
- Bottom nav + sidebar active states
- Closing screen-specific sheets on navigate away (calendar sheet, accounts sheets, category picker)
- Resetting header actions per screen (Plan header, Accounts header)
- Sidebar notes switching

## Mock Data — Alex, Feb 2026
All screens use a consistent dataset for a single fictional user:
- **Name:** Alex, 26, Melbourne
- **Income:** $6,480/month net (fortnightly $3,240 from Acme Corp; paydays Feb 8 + Feb 22)
- **Accounts:** ANZ Everyday $1,847 · ANZ NetSaver $12,450 (3.75% p.a.) · ANZ Credit Visa −$843
- **Net Position:** +$13,454
- **Today:** Thu 19 Feb 2026 (day 11 of 14-day pay cycle)
- **Feb spend so far:** $2,781 total ($146/day avg, 19 days)
- **Top categories:** Housing $1,600 · Dining Out $341 · Groceries $178 · Transport $122
- **Savings rate:** 18% this month (Jan was 27%; Dec was 10% due to holidays)
- **Lifestyle Creep:** +6% delta (spend +11% vs income +5% over 3mo vs 12mo)
- **Safety Net:** 2.6 months (amber — $2,100 short of 3-month target)
- **AI milestone:** 24/30 transactions reviewed (80%)
- **Goals:** Japan Trip $2,840/$4,200 (ETA May 2026) · Safety Net $12,450/$14,550 (ETA Apr 2026) · New Car $3,100/$10,000 (ETA Dec 2026)

## End-of-Session Documentation Protocol

**When the user says "update docs" (or equivalent) at the end of a session, ALL of the following must be written or updated before the session closes:**

1. **CLAUDE.md** — any new architectural decisions, screen summaries, metric definitions, JS function names, state variables, data flow notes, or mock data changes made during the session must be reflected here
2. **Per-screen `feature-list.md`** — any new features, formula changes, edge case discoveries, or calculation schedule changes for any screen touched this session
3. **Per-screen `design-spec.md`** — any new components, layout changes, animation specs, colour usage, or interaction patterns introduced
4. **Per-screen `database-schema.md`** — any new tables, columns, indexes, or data flow steps added
5. **New screen folders** — if a new screen was started or completed, its full `Accounts/`-style folder with all four files must be created before session end
6. **"Pending Documentation" section below** — keep this list accurate; remove items as they're completed, add items for anything deferred

The goal is zero documentation debt at the end of every session. A future Claude instance reading only this repo should be able to reconstruct exactly what was built and why.

---

## Pending Documentation

All screen documentation is now complete. If new features, screens, or architectural decisions are made in future sessions, update docs via the End-of-Session Protocol above.
