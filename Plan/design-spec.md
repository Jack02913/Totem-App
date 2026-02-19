# Totem — Plan Screen Design Specification
## Goals + Budgets: Visual System, Component Rules & Layout Proportions

> **Reference file:** See `plan-mockup.html` for a live rendered example of every component described here.
> **Platform targets:** iOS (React Native), Android (React Native), Web (React). Mobile is primary.
> **Design system:** Extends `Dashboard/design-spec.md`. All colour tokens, typography scale, spacing system, and base component rules apply here — this document covers Plan-screen-specific additions and overrides only.

---

## 1. Design Principles (Plan-specific)

The Plan screen serves a different emotional register than the Dashboard. Where the Dashboard answers "where am I right now?", Plan answers "where am I going, and am I on track?". This changes the visual emphasis.

1. **Progress is the hero.** Every goal card shows progress at a glance. The user should feel forward momentum — even small progress should feel visible and satisfying.

2. **Goal type is immediately legible.** The three goal types (Save Up, Spending Habit, Debt Paydown) have distinct visual identities. A user who builds multiple goals should never need to read the goal name to know which type it is.

3. **Streaks should feel earned.** The streak indicator for Spending Habit goals is the most emotionally resonant element on the screen. When a streak is active, it should feel like something worth protecting. When it's absent, its absence should motivate.

4. **AI suggestions should feel like advice, not ads.** The AI suggestion card uses the same visual language as the Dashboard AI Insight card — calm, data-backed, positioned as guidance rather than a prompt. The "Totem AI" branding makes the source clear.

5. **Budgets is a management tool.** The Budgets sub-tab is not motivational — it's operational. Its visual treatment is deliberately denser and more functional. The same burn-rate bar from the Dashboard is reused here but in a list context designed for editing and monitoring.

---

## 2. Screen Architecture

### Layout Shell
```
┌────────────────────────────────────┐
│  Status bar + back/header          │  ~56px
│  "Plan" title  [Goals] [Budgets]   │  sub-tab bar ~44px
├────────────────────────────────────┤
│                                    │
│  Tab content (scrollable)          │  flex: 1
│  bottom padding: 96px              │  clears bottom nav + FAB
│                                    │
├────────────────────────────────────┤
│  Floating AI Chat button (FAB)     │  fixed, above bottom nav
│  BOTTOM NAVIGATION   82px fixed   │
└────────────────────────────────────┘
```

### Sub-Tab Bar
```
Layout:   horizontal flex, two equal segments
Height:   44px
Divider:  bottom border 1px solid var(--border)

Active tab:
  font-size: 14px, weight 700, color var(--text)
  bottom border: 2px solid var(--brand), border-radius: 2px 2px 0 0

Inactive tab:
  font-size: 14px, weight 500, color var(--text-2)

Tab switch animation:
  Sliding underline: transform: translateX(), 250ms, ease
```

---

## 3. Goal Type Visual Identity System

Each goal type has a unique colour and icon convention. This system ensures instant recognition.

| Goal Type | Left-border colour | Icon background | Icon |
|---|---|---|---|
| Save Up | `--brand` (#7C6EFA) | `--brand-light` | 🎯 or user-selected emoji |
| Spending Habit | `--teal` (#4ECDC4) | `--teal-light` | 🔥 streak (or category emoji) |
| Debt Paydown | `--green` (#51CF66) | `--green-light` | 📉 or 💳 |

### Left Border Rule
Every goal card has a 3px left border in the goal type's accent colour. This is the primary visual differentiator and must not be removed. When cards are displayed in a mixed list, the left border creates an immediately scannable colour-coded index.

```css
/* Applied to all goal cards */
border-left: 3px solid var(--type-accent);

/* Save Up */
--type-accent: var(--brand);

/* Spending Habit */
--type-accent: var(--teal);

/* Debt Paydown */
--type-accent: var(--green);
```

---

## 4. Component Specifications

### 4.1 Page Header
```
Layout: flex row, space-between, align-items: center
Padding: 16px 20px 8px
Height: ~56px

Left: Page title
  font-size: 22px, weight: 800, letter-spacing: -0.5px, color: var(--text)

Right: Add button (Goals tab) or Edit button (Budgets tab)
  "+ Add" / "Edit" link: 14px, weight: 600, color: var(--brand)
```

### 4.2 Sub-Tab Bar
```
Container:
  padding: 0 20px
  border-bottom: 1px solid var(--border)

Each tab:
  padding: 10px 0
  margin-right: 24px (tabs are left-aligned, not full-width equal segments)

Active indicator:
  position: absolute, bottom: 0
  width: matches tab text width
  height: 2px
  background: var(--brand)
  border-radius: 2px 2px 0 0
  transition: left 250ms ease, width 250ms ease
```

### 4.3 30-Transaction Milestone Card
Shown in Goals tab when AI is not yet activated. Replaces or precedes the AI suggestion card.

```
Background: var(--card)
Border: 1px solid rgba(124,110,250,0.2)
Border-radius: 16px
Padding: 16px

Top row:
  Icon (🤖): 36×36px container, border-radius 10px, background var(--brand-light)
  Label: "TOTEM AI" — 10px, weight 700, uppercase, letter-spacing 1px, var(--brand)
  Title: "Activate Totem AI" — 14px, weight 700, var(--text)

Progress bar:
  Standard bar track (8px height — slightly taller than standard 6px for emphasis)
  Fill: var(--brand) gradient
  Margin: 10px 0

Progress label row (flex, space-between):
  "X of 30 transactions reviewed" — 12px, var(--text-2)
  "X more to go" — 12px, weight 600, var(--brand)

Sub-text:
  "Review more transactions to unlock AI goal suggestions and insights."
  12px, var(--text-2), line-height: 1.5
  margin-top: 8px

CTA:
  "Go to Review →" — 13px, weight 600, var(--brand), margin-top 10px
```

### 4.4 AI Suggestion Card
Shown when AI is active and a pending suggestion exists.

```
Background: var(--card)
Border: 1px solid rgba(124,110,250,0.25)  // brand glow — suggests AI origin
Border-radius: 16px
Padding: 16px

Top row (flex, gap: 12px):
  Icon container: 36×36px, border-radius 10px, background var(--brand-light)
    Sparkle emoji (✨): 17px
  Text block:
    Tag: "TOTEM AI" — 10px, weight 700, uppercase, letter-spacing 1px, var(--brand)
    Goal type label: "Spending Habit Goal" — 12px, var(--text-2)

Insight text:
  font-size: 14px, weight: 600, color: var(--text)
  line-height: 1.5
  margin: 10px 0 4px

Preview text (calculation summary):
  font-size: 12px, color: var(--text-2), line-height: 1.5
  background: rgba(255,255,255,0.04)
  border-radius: 8px
  padding: 8px 10px
  margin-bottom: 12px

Action row (flex, gap: 8px):
  [Review Suggestion]: primary pill button
    background: var(--brand), color: white
    padding: 8px 16px, border-radius: 20px
    font-size: 13px, weight 600

  [Not Now]: ghost button
    background: transparent, color: var(--text-2)
    padding: 8px 12px, border-radius: 20px
    font-size: 13px, weight 500
    border: 1px solid var(--border)
```

### 4.5 Goal Card (Base)

All goal types share this base card structure with type-specific sections inserted.

```
Container:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-left: 3px solid var(--type-accent)  // goal type colour
  Border-radius: 16px
  Padding: 16px
  Margin-bottom: 10px
  overflow: hidden

Header row (flex, space-between, align-items: flex-start):
  Left: Icon + title block
    Icon container: 34×34px, border-radius 9px, background: var(--type-accent-light)
      Emoji: 16px
    Title: 15px, weight 700, var(--text)
    Subtitle: 11px, var(--text-2), margin-top 1px  // "Save Up Goal" / "Spending Habit" / "Debt Paydown"

  Right: Status badge (see §4.11)
    + optional streak indicator (Spending Habit only — see §4.7)

Progress section (see per-type rules below)
```

### 4.6 Save Up Goal Card Body
```
// Below header
Progress row (flex, space-between, margin: 12px 0 6px):
  Amount section:
    Current: 22px, weight 800, letter-spacing -0.8px, var(--text)
    " of " $target: 13px, var(--text-2)
  Percentage: 16px, weight 700, var(--brand), text-align: right

Progress bar: standard 6px track, fill var(--brand)
  No period marker on Save Up — linear progress only

Footer row (flex, space-between, margin-top: 10px):
  Left:
    "Projected:" label: 11px, var(--text-2)
    Date/status: 12px, weight 600, var(--text)
  Right: status badge (AHEAD / ON TRACK / AT RISK / BEHIND / IN PROGRESS)

// If auto-transfer enabled:
Auto-transfer pill (margin-top: 8px):
  background: rgba(255,255,255,0.04), border-radius 20px
  padding: 4px 10px, display: inline-flex, gap: 6px
  "↻ Auto-transfer: $X/mo": 11px, var(--text-2)
```

### 4.7 Spending Habit Goal Card Body
Totem's signature goal type. The card has the most visual richness.

```
// Below header
// Period progress section (current month)

Month label row (flex, space-between, margin-top: 12px):
  "February" — 11px, weight 600, var(--text-2)
  "X days left" — 11px, var(--text-3)

Amount row (flex, space-between, margin: 4px 0):
  Spent: 22px, weight 800, letter-spacing -0.8px, var(--text)
  " of " $target/mo: 13px, var(--text-2)

Progress bar: 6px track
  Includes period marker (same as Dashboard Burn Rate)
    -- white vertical tick at current-day position
  Fill colour: fill-green (on track) / fill-amber (at risk) / fill-red (over)
  Margin-bottom: 4px

Bar labels (flex, space-between):
  Status text: 11px, var(--text-2)  // "On track · $X remaining"
  Variance: 11px, weight 600, var(--green) or var(--red)

// Streak indicator row (only shown if current_streak >= 1)
Streak row (margin-top: 10px, padding-top: 10px, border-top: 1px solid var(--border)):
  Layout: flex, align-items: center, gap: 6px
  Fire emoji (🔥): 14px
  Streak text: 12px, weight 600
    1 month: var(--text-2)  // "1 month ✓" — muted, not yet exciting
    2-3 months: var(--amber)  // building momentum
    4+ months: var(--brand)  // a real streak — brand colour highlight
  Longest streak: 11px, var(--text-3), margin-left: auto  // "Best: X"
```

**Rationale for streak colour progression:** The streak builds from muted (1 month) to amber (2-3) to brand purple (4+). This mirrors how habits actually feel — the first month is uncertain, a few months builds momentum, and 4+ months is a genuine behavioural pattern worth celebrating with the brand's primary colour.

### 4.8 Debt Paydown Goal Card Body
```
// Below header
Progress row (margin-top: 12px):
  Cleared amount: 22px, weight 800, letter-spacing -0.8px, var(--green)
  " paid off" label: 13px, var(--text-2)

Remaining row:
  "$X remaining" — 13px, var(--text-2), margin-top 2px

Progress bar: 6px track, fill var(--green)
  No period marker — continuous progress, not monthly

Footer row (flex, space-between, margin-top: 10px):
  Left:
    "Payoff projected:" — 11px, var(--text-2)
    Date: 12px, weight 600, var(--text)
  Right: status badge

// If interest_rate is set:
Interest saved pill (margin-top: 8px):
  background: rgba(81,207,102,0.08), border-radius 20px
  padding: 4px 10px, display: inline-flex
  "💰 Saving ~$X in interest vs. minimum payments": 11px, var(--green)
```

### 4.9 Completed Goals Section
```
Collapsed header row:
  "COMPLETED" — 11px, weight 700, uppercase, letter-spacing 0.8px, var(--text-2)
  Count badge: pill, var(--brand-light)/var(--brand), "X goals"
  Chevron: 16px, var(--text-3), rotates on expand

When expanded:
  Completed goal cards appear with reduced opacity (0.6)
  Left border: var(--text-3) — desaturated, no longer type-colour
  Status badge: "COMPLETED" (green) replaces the progress badge
  No streak indicator, no progress bar fill animation
```

### 4.10 Empty State (No Goals)
```
Centred in available space, padding: 40px 24px

Illustration placeholder:
  80×80px circle, background: var(--card)
  "🎯" emoji: 36px

Heading: "No goals yet"
  18px, weight 700, var(--text), margin-top: 16px

Sub-text: "Goals help you build better money habits — save toward something, cut a spending category, or knock out debt."
  14px, var(--text-2), line-height: 1.6, text-align: center, margin-top: 8px

CTA button: "+ Create your first goal"
  full-width, 48px height, border-radius 12px
  background: var(--brand), color: white
  font-size: 15px, weight 700
  margin-top: 24px
```

### 4.11 Status Badges (Goal-specific)

Extends the Dashboard badge system with goal-specific variants.

| Badge | Colours | Usage |
|---|---|---|
| AHEAD | green | Save Up / Debt Paydown: projected ahead of target date |
| ON TRACK | green | All types: on track for current period |
| AT RISK | amber | Spending Habit: spending faster than safe pace |
| OVER | red | Spending Habit: already exceeded target this month |
| BEHIND | red | Save Up / Debt Paydown: falling behind target date |
| IN PROGRESS | brand | No target date set — no deadline pressure |
| COMPLETED | green | Goal finished |
| PAUSED | var(--text-3) on transparent | Goal manually paused |

### 4.12 Floating AI Chat Button (FAB)
This button appears on every screen, fixed position above the bottom nav.

```
Size: 52×52px circle
Background: var(--brand)
Shadow: 0 4px 20px rgba(124,110,250,0.5)
Position: fixed, bottom: 96px (above 82px nav + 14px gap), right: 20px

Icon: ✨ sparkle emoji, 22px
  — or the Totem "T" logomark when branding is finalised

Animation on app load:
  Scale from 0.6 → 1.0 with spring easing, 400ms
  Delay: 800ms (after content loads, so it doesn't distract from main content)

Tap: opens AI Chat overlay (slides up from bottom, full-screen, dismissible)

Unread indicator (when AI has a proactive message):
  8×8px circle, top-right corner of FAB
  background: var(--amber)
  border: 2px solid var(--bg)
```

### 4.13 Add Goal Bottom Sheet — Type Selector
The first step when creating a new goal. This sheet slides up from the bottom.

```
Sheet:
  Background: var(--card)
  Border-radius: 20px 20px 0 0
  Padding: 8px 20px 32px
  max-height: 80vh, scrollable

Handle bar:
  32×4px, background: rgba(255,255,255,0.2), border-radius 2px
  Centred, margin: 12px auto 20px

Title: "What type of goal?"
  17px, weight 800, letter-spacing -0.5px, margin-bottom: 6px

Sub-text: "Choose the type that fits what you're trying to do."
  13px, var(--text-2), margin-bottom: 20px

// Three option cards (stacked, not horizontal — names need room)
Each option card:
  Background: rgba(255,255,255,0.04)
  Border: 1px solid var(--border)
  Border-left: 3px solid var(--type-accent)  // goal type colour
  Border-radius: 12px
  Padding: 14px 16px
  Margin-bottom: 10px
  Layout: flex row, gap: 14px, align-items: center

  Icon container: 40×40px, border-radius 10px, background var(--type-accent-light)
    Emoji: 20px

  Text block:
    Type name: 15px, weight 700, var(--text)
    Description: 12px, var(--text-2), line-height: 1.4, margin-top: 2px

  Chevron: 16px, var(--text-3), right side

Type cards content:
  Save Up:
    icon: 🎯, name: "Save Up", desc: "Set a target amount and track your progress toward it"
  Spending Habit:
    icon: 📊, name: "Reduce Spending", desc: "Set a monthly limit on a spending category and build a streak"
  Debt Paydown:
    icon: 💳, name: "Pay Off Debt", desc: "Track progress on clearing a credit card or loan"
```

---

## 5. Budgets Sub-Tab

### 5.1 Period Context Banner
```
Background: rgba(255,255,255,0.04)
Border: 1px solid var(--border)
Border-radius: 12px
Padding: 10px 14px
Margin-bottom: 12px
Layout: flex, space-between, align-items: center

Left text: "Day 19 of February"
  13px, weight 500, var(--text)

Right text: "63% of month elapsed"
  12px, var(--text-2)

Period marker indicator line:
  Small inline visual — thin 1px vertical line in var(--brand), 10px tall
  Appears between the two text elements
  Same conceptual marker as the burn rate bar tick
```

### 5.2 Budget Summary Header
```
Layout: grid, 2 columns, gap: 10px
Margin-bottom: 16px

Total Budgeted card:
  Background: var(--card), border: 1px solid var(--border), border-radius: 12px, padding: 14px
  Label: "TOTAL BUDGETED" — 10px, uppercase, var(--text-2)
  Amount: 22px, weight 800, letter-spacing -0.8px, var(--text)

Total Spent card:
  Same styling
  Amount colour: var(--red) if over sum, var(--text) otherwise
  Label: "SPENT THIS MONTH"
```

### 5.3 Budget Row
Each category with an active budget. Extends the Dashboard Budget Burn Rate item.

```
Container (list):
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 16px
  overflow: hidden

Each row:
  Padding: 14px 16px
  Border-bottom: 1px solid var(--border) (except last)
  Tap area: full row, opens inline edit

  Top row (flex, space-between):
    Left: emoji + category name
      Emoji: 18px
      Name: 14px, weight 600, var(--text)
      // If linked to a goal: small teal dot after name (tooltip: "Synced with goal")
    Right: spent / budget
      Spent: 14px, weight 700, var(--text)
      " / $budget": 11px, var(--text-2)

  Progress bar: 6px, with period marker tick (same as Dashboard)

  Bottom row (flex, space-between):
    Status badge (On Track / At Risk / Over)
    Percentage: 11px, var(--text-3)

  // Inline edit mode (tap amount):
  Budget amount field:
    Position: replaces the "/ $budget" display
    Input: underline style, var(--brand) text
    Done on blur or checkmark tap
    No navigation — edit in place
```

### 5.4 Add Budget Button
```
Full width, margin-top: 12px
Height: 48px
Background: transparent
Border: 1px dashed rgba(124,110,250,0.3)
Border-radius: 12px
Color: var(--brand)
Font: 14px, weight 600

Content: "+ Add Budget"
```

### 5.5 Budget-Goal Sync Indicator
When a budget is linked to a Spending Habit goal:
```
// In the budget row, after category name:
Small dot: 6×6px, background: var(--teal), border-radius: 50%, margin-left: 6px
// Long press or info tap: shows tooltip "Synced with Dining Out goal · $464/mo"

// In the goal card (Spending Habit), below the progress bar:
Linked budget pill:
  background: rgba(78,205,196,0.08), border-radius 20px, padding: 3px 10px
  "⟷ Linked to Dining budget": 11px, var(--teal)
```

---

## 6. Screen Layout & Proportions (Mobile — 390×844px)

### Goals Tab — Viewport Priority Order

| Element | Approx. Height | Notes |
|---|---|---|
| Page header + sub-tab bar | ~100px | Fixed — always visible |
| AI suggestion card (or milestone card) | ~140px | P0 when AI active — primary CTA |
| First goal card | ~120–140px | P1 — most recent / most urgent |
| Second goal card | ~120–140px | — |
| Third goal card (partially visible) | ~60px visible | Scroll affordance |

**The first goal card should always be partially visible above the fold** (the scroll area begins scrolling upward) without requiring the user to scroll to confirm that there is content. The AI suggestion card + one full goal card + top of a second goal card should fill the viewport.

### Goals Tab — Card Heights by Type

| Goal Type | Card Height (approx) | Tall state (streak visible) |
|---|---|---|
| Save Up | 120px | — |
| Spending Habit | 130px | 155px (streak row adds 25px) |
| Debt Paydown | 120px | — (no streak) |

### Budgets Tab — Viewport Priority Order

| Element | Approx. Height | Notes |
|---|---|---|
| Page header + sub-tab bar | ~100px | |
| Period context banner | ~44px | |
| Budget summary 2-col | ~80px | |
| Section header ("BUDGETS") | ~28px | |
| Budget rows (5–6 visible before scroll) | ~65px each | |
| Add Budget button | ~60px | |

The Budgets tab prioritises density over visual richness. 5-6 budget rows should be visible without scrolling on a standard screen — this allows quick scanning and editing without constant scrolling.

---

## 7. Animations & Transitions (Plan-specific)

| Interaction | Animation | Duration | Easing |
|---|---|---|---|
| Tab switch (Goals ↔ Budgets) | Content crossfade + underline slide | 250ms | ease |
| Goal card expand (completed section) | Height from 0 + opacity 0→1 | 300ms | ease-out |
| Bottom sheet slide-up | `transform: translateY(100% → 0)` | 350ms | cubic-bezier(0.25, 0.46, 0.45, 0.94) |
| Backdrop dim | Opacity 0→0.6 | 350ms | ease |
| Progress bar fill (on card render) | Width from 0 | 700ms | ease-out |
| Streak fire emoji | Pulse scale (1 → 1.1 → 1) | 1s | ease-in-out, loops on milestone |
| FAB entrance | Scale 0.6 → 1 with bounce | 400ms, 800ms delay | spring |
| Goal completion celebration | Full-card confetti burst | 1.2s | — |

### Month Close Celebration
When a Spending Habit goal closes successfully at month end, the goal card plays:
```
1. Green flash behind card (rgba(81,207,102,0.2) briefly)
2. Confetti burst from top of card (CSS-only, 20–30 coloured particles)
3. Streak number increments with count-up animation
4. Toast notification slides in from top: "🎉 February goal hit! 3 months in a row."
```

---

## 8. Web (Desktop) Adaptation

The Plan screen adapts to desktop with a 2-column layout.

```
┌─────────────────────────────────────────────────────────┐
│  Left sidebar nav (Goals, Budgets links)                │
├──────────────────────────────┬──────────────────────────┤
│  Goals list (left col)       │  Selected goal detail    │
│  - AI suggestion card        │  (right panel)           │
│  - Goal cards (condensed)    │  Full goal detail:       │
│  - Add Goal                  │  - Progress chart        │
│                              │  - Monthly history       │
│                              │  - Edit controls         │
└──────────────────────────────┴──────────────────────────┘
```

On desktop:
- Goal cards in the left column are condensed (no body, just header + progress bar + badge)
- Selecting a goal opens the full detail view in the right panel
- Budgets tab shows a table layout with inline editing

---

## 9. Empty States

| State | Display |
|---|---|
| No goals | Centred empty state with 🎯, heading, sub-text, full-width CTA button (§4.10) |
| No budgets | Empty state with 📊, "Set spending limits to track where your money goes" + Add Budget CTA |
| No AI suggestions (AI active, all dismissed) | Card hidden — no empty suggestion card. Just goals list. |
| AI not yet activated (< 30 transactions) | 30-transaction milestone card (§4.3) shown at top of Goals tab |
| Goal paused | Card shown at reduced opacity, "PAUSED" badge, "Resume" button replaces progress |

---

## 10. Accessibility Notes

- Goal type is not conveyed by colour alone — the subtitle ("Save Up Goal", "Spending Habit", "Debt Paydown") and left-border colour work together. The type label is always present for screen readers.
- Progress bars include `aria-valuenow`, `aria-valuemin`, `aria-valuemax`, `aria-label="Goal progress: X%"`.
- Streak indicator: screen reader reads as "3-month streak" not just the fire emoji.
- Bottom sheet: focus trap on open, ESC dismisses, return focus to trigger on close.
- Inline budget editing: field should be labelled `aria-label="Budget amount for [category name]"`.
- FAB: `aria-label="Open AI Chat"`, `role="button"`.
