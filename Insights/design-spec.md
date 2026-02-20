# Totem — Insights Design Specification
## Visual System, Component Rules & Layout Proportions

> **Reference file:** See `app-mockup.html` (all `ins-` prefixed IDs and functions) for a live rendered example of every component described here.
> **Platform targets:** iOS (React Native), Android (React Native), Web (React). Mobile is primary.

---

## 1. Design Principles (Insights Screen)

1. **Context before conclusion.** Every chart and metric on this screen exists to explain the numbers on the Dashboard — not to repeat them. If a metric is on the Dashboard, the Insights screen should show why it is what it is.

2. **Time is the primary axis.** Unlike the Dashboard (which is "right now"), Insights is about patterns over time. Every component should make temporal context clear — which period, how many months, whether data is partial.

3. **Charts earn their space.** A bar chart with no insight is just decoration. Every visualisation should have a clear takeaway legible in under 3 seconds — use colour, labels, and captions to make the insight explicit.

4. **Forecasts feel probabilistic, not certain.** Dashed lines, lighter fills, and "projected" labels communicate that future data is an estimate. Users should feel informed, not misled.

---

## 2. Tab Navigation (Spend | Trends | Income | Forecast)

Same pattern as Plan and Accounts tab bars. Controlled by `switchInsTab(tab)`.

```
Tab bar:
  Position: below screen header, sticky
  Height: 44px
  Background: var(--card)
  Border-bottom: 1px solid var(--border)

Each tab:
  flex: 1
  font-size: 13px  // slightly smaller — 4 tabs fit tighter
  font-weight: 600
  text-align: center
  padding: 12px 0
  color: var(--text-2)

Active tab:
  color: var(--brand)
  border-bottom: 2px solid var(--brand)
  margin-bottom: -1px
```

---

## 3. Period Chip Selector (Spend Tab)

```
Container:
  display: flex, gap: 8px
  padding: 16px 20px 8px
  overflow-x: auto (in case of many chips on small screens)
  -webkit-overflow-scrolling: touch

Each chip:
  padding: 6px 16px
  border-radius: 20px (full pill)
  font-size: 13px, font-weight: 600
  cursor: pointer
  white-space: nowrap

Inactive chip:
  background: var(--card)
  border: 1px solid var(--border)
  color: var(--text-2)

Active chip:
  background: var(--brand-light)
  border: 1px solid rgba(124,110,250,0.4)
  color: var(--brand)

Transition: background + border-color + color, 150ms ease

// Three chips: "This Month", "3 Months", "12 Months"
// Tapping a chip recalculates all Spend tab metrics for that period
```

---

## 4. KPI Row

Three headline stats in a horizontal row. Used on both the Spend tab and Income tab.

```
Container:
  display: grid
  grid-template-columns: repeat(3, 1fr)
  gap: 8px
  padding: 8px 20px 16px

Each KPI cell:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 14px
  Padding: 12px 10px
  text-align: center

Label:
  font-size: 10px, weight 700, uppercase, letter-spacing 0.7px
  color: var(--text-2)
  margin-bottom: 4px

Value:
  font-size: 20px, weight 800, letter-spacing: -0.8px
  color: var(--text)

Modifier (small sub-label below value, optional):
  font-size: 10px, color var(--text-2) or var(--green) / var(--red)
  // e.g. "+5%" in green for the Income Growth KPI
```

---

## 5. Category Bar Component

Used in the Spend tab category breakdown. Each bar is a full-width row.

```
Outer container:
  padding: 0 20px
  // rows stacked, no card wrapper — flow directly in the content area

Each category row:
  margin-bottom: 12px

Header row (flex, space-between):
  Left: emoji (16px) + category name (13px, weight 600, var(--text))
  Right: amount + pct-of-income (13px, weight 600, right-aligned)
    Amount: var(--text)
    Pct: var(--text-2), smaller (11px)  // e.g. "$341 · 5.3% of income"

Bar track:
  width: 100%
  height: 6px
  background: rgba(255,255,255,0.08)
  border-radius: 4px
  margin-top: 6px
  overflow: hidden

Bar fill:
  height: 100%
  border-radius: 4px
  width: (category_spend / reference_income) × 100%  // capped at 100%
  transition: width 0.6s ease-out (on period change)

Fill colours:
  Default:              var(--brand)
  Over budget:          var(--red)
  > 30% of income:      var(--amber)

Over-budget indicator:
  If category is over budget: small red badge "Over" appears right of bar
  badge-red spec from Dashboard design-spec
```

---

## 6. Merchant Bar Component

Used in the Spend tab Top 5 merchants section.

```
Section header: standard section header row (11px uppercase label + count)

Container:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 16px
  Padding: 4px 0
  overflow: hidden

Each merchant row:
  Padding: 11px 16px
  Border-bottom: 1px solid var(--border) (except last)

Header (flex, space-between):
  Left: merchant name (13px, weight 600, var(--text))
  Right: amount (13px, weight 700, var(--text))

Fill bar:
  height: 4px  // thinner than category bars — secondary metric
  margin-top: 6px
  background: var(--brand-light)  // lighter fill for merchant bars
  border-radius: 2px

  Fill: (merchant_total / top_merchant_total) × 100%
  Fill colour: var(--brand)
  Top merchant: always 100% (anchor)
```

---

## 7. Dual Bar Chart (Income Tab)

Side-by-side bars per month showing income vs spend. Used in Income tab.

```
Chart container:
  padding: 0 20px
  overflow-x: hidden

Monthly group:
  display: inline-flex (or grid column)
  flex-direction: column
  align-items: center
  width: (container_width - 40px) / 5  // 5 months, equal width

Within each month group:
  Bar pair wrapper:
    display: flex, gap: 3px, align-items: flex-end
    height: 80px  // fixed chart height

  Income bar:
    width: ~18px
    background: var(--brand)
    border-radius: 3px 3px 0 0
    height: (monthly_income / max_income) × 80px

    Partial month variant:
      background: transparent
      border: 2px dashed var(--brand)
      // dashed outline = income not fully received

  Spend bar:
    width: ~18px
    background: var(--red)
    border-radius: 3px 3px 0 0
    height: (monthly_spend / max_income) × 80px
    // Scaled against max_income (same reference as income bars)

  Month label:
    font-size: 10px, color var(--text-2)
    text-align: center, margin-top: 6px

Chart legend (below chart):
  inline-flex, gap: 16px
  "■ Income" (brand) + "■ Spend" (red)
  font-size: 11px, weight 600
```

---

## 8. Lifestyle Creep Expanded Card (Trends Tab)

Full breakdown of the Lifestyle Creep metric. Larger than the dashboard summary.

```
Card:
  Background: var(--card)
  Border: 1px solid var(--amber) at 30% opacity (amber glow — watch/detected states)
         OR var(--border) (balanced/improving state)
  Border-radius: 16px
  Padding: 16px
  Margin: 0 20px 12px

Header row:
  Left: "LIFESTYLE CREEP" (11px, uppercase, var(--text-2))
  Right: status badge
    'improving': badge-brand    "Improving ↓"
    'balanced':  badge-green    "Balanced"
    'watch':     badge-amber    "Watch This"
    'detected':  badge-red      "Detected ⚠"

Delta figure:
  font-size: 32px, weight 800, letter-spacing: -1.5px
  Prefix sign: always shown (+ or -)
  Colour: same as status (green/amber/red/brand)
  Sub: "spend growing faster than income" or equivalent contextual label

Breakdown rows (2):
  Each row:
    display: flex, space-between
    padding: 8px 0
    border-top: 1px solid var(--border)

  Left: metric label (13px, var(--text-2))  e.g. "Spending growth (3mo vs 12mo)"
  Right: percentage (13px, weight 700, var(--text))  e.g. "+11%"

Driver row:
  Left: "Main driver" (11px, var(--text-2))
  Right: category name + badge  (e.g. "🍔 Dining Out" + amber badge "+34%")

Building state (< 3 months data):
  Replace delta figure with progress bar:
    "Building baseline: X of 3 months complete"
    Standard 6px progress bar (fill-brand)
```

---

## 9. Safety Net Detail Card (Trends Tab)

```
Card:
  Same base as standard card
  Border: var(--amber-light) border if below target; var(--green-light) if healthy

Header:
  "SAFETY NET" (label) + months value (28px, weight 800) + status badge

Breakdown:
  Liquid savings row:    "Liquid savings" + "$12,450"
  Avg expenses row:      "3-month avg expenses" + "$4,788/mo"
  Shortfall row:         "To reach 3-month target" + "$2,100" (amber)

Progress bar:
  Track height: 8px
  Fill: (safety_net_months / target_months) × 100%
  Fill colour: matches status (green/amber/red)
  Target marker (thin white line at 100%):
    width: 2px, height: 14px (extends above and below track)
    background: rgba(255,255,255,0.4)
    position: absolute, left: 100%  // at the target end
    label below: "3 months" (10px, var(--text-2))
```

---

## 10. 5-Bar Spend Trend Chart (Trends Tab)

```
Chart container:
  display: flex, align-items: flex-end, gap: 6px
  height: 80px (bar area only)
  padding: 0 20px

Each bar column:
  flex: 1
  display: flex, flex-direction: column, align-items: center, gap: 6px

Bar:
  width: 100%
  border-radius: 4px 4px 0 0
  height: (monthly_spend / max_monthly_spend) × 80px

  Colours:
    Highest spend month:  var(--red)      // Dec — holiday spike
    Current partial:      diagonal stripe or dotted border (see below)
    All others:           var(--brand)

Partial month bar (current):
  Standard bar fill + dashed overlay border
  Opacity: 0.7 (slightly lighter to signal incompleteness)
  // OR: split bar: solid portion = actual spend, dashed portion = projected remainder

Bar value label (above each bar):
  font-size: 10px, weight 600, color var(--text-2)
  "$X,XXX" format

Month label (below bar):
  font-size: 10px, color var(--text-2)
  Current month: var(--brand) colour, asterisk suffix "Feb*"

Note below chart:
  "*Feb data is partial (19 of 28 days)" (11px, var(--text-2), italic or muted)
```

---

## 11. Forecast Hero Card (Forecast Tab)

```
Card:
  Background: linear-gradient(135deg, #1b1244 0%, #141929 65%)  // same as dashboard hero
  Border: 1px solid rgba(124,110,250,0.25)
  Border-radius: 20px
  Padding: 20px
  Margin: 16px 20px 8px

Label: "MONTH-END PROJECTION" (11px, uppercase, var(--brand), letter-spacing 1px)

Hero amount:
  "+$2,241" (40px, weight 800, letter-spacing -2px)
  Colour: var(--green) (positive surplus) or var(--red) (projected deficit)
  Prefix $ at 20px, weight 600, var(--text-2), valign top

Sub-line:
  "Income $6,480 · Projected spend $4,239"
  (12px, var(--text-2), margin-top 6px)

Status badge:
  "On Track" badge-green
  OR "At Risk" badge-amber
  OR "Overspend Risk" badge-red
  (same logic as Dashboard month end projection)
```

---

## 12. 90-Day Cash Flow Chart (Forecast Tab)

Three paired bars (income vs projected spend) for Mar, Apr, May.

```
Chart container:
  display: grid, grid-template-columns: repeat(3, 1fr), gap: 12px
  padding: 0 20px

Each month column:
  Text-align: center

  Bar pair (income + spend side by side):
    display: flex, gap: 3px, align-items: flex-end, justify-content: center
    height: 70px

    Income bar:
      width: 22px
      background: var(--brand)  (lighter shade: 60% opacity)
      border: 1px dashed var(--brand)  // dashed = projected, not actual
      border-radius: 3px 3px 0 0

    Spend bar:
      width: 22px
      background: var(--red)  (60% opacity)
      border: 1px dashed var(--red)
      border-radius: 3px 3px 0 0

  Surplus label (below bars):
    font-size: 11px, weight 700
    color: var(--green) if surplus; var(--red) if deficit
    e.g. "+$2,380"

  Month label:
    font-size: 11px, color var(--text-2)
    "Mar", "Apr", "May"

Note below chart:
  "Projections based on current spending and expected income" (11px, var(--text-2))
```

---

## 13. Goal ETA Card (Forecast Tab)

```
Container:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 16px
  Overflow: hidden
  Margin: 0 20px 12px

Each goal row:
  Padding: 14px 16px
  Border-bottom: 1px solid var(--border) (except last)

  Goal name: 14px, weight 600, var(--text)
  Progress bar (4px height, fill-brand):
    width: (current / target) × 100%
    margin: 6px 0

  Bottom row (flex, space-between):
    Left: "ETA May 2026" (12px, weight 600, var(--brand))
    Right: "68%" (12px, weight 700, var(--text)) + progress pct
```

---

## 14. Risk Flag Component (Forecast Tab)

A list of 2–5 forward-looking flags. Each flag is a row card.

```
Each flag card:
  Background: var(--card)
  Border-radius: 12px
  Padding: 12px 14px
  Margin: 0 20px 8px
  Display: flex, gap: 12px, align-items: flex-start

Severity dot:
  width/height: 8px, border-radius 50%
  margin-top: 4px  // aligns with first line of text
  Colours:
    'spend_risk':       var(--red)
    'known_expense':    var(--amber)
    'goal_opportunity': var(--green)
    'milestone_close':  var(--brand)

Content:
  Title: 13px, weight 600, var(--text)
  Body: 12px, weight 500, var(--text-2), margin-top 3px, line-height 1.5

Left border accent:
  4px solid left border (colour matches severity dot)
  // Visual shorthand: coloured left edge → users learn to scan left side for severity
  border-radius: 12px on right side, 0 on left (or use box-shadow inset)
```

---

## 15. Screen Layout & Proportions (Mobile — 390×844px)

```
┌────────────────────────────────────┐
│  Status bar              54px      │
├────────────────────────────────────┤
│  Screen header           56px      │  "Insights"
├────────────────────────────────────┤
│  Tab bar                 44px      │  Spend | Trends | Income | Forecast
├────────────────────────────────────┤
│                                    │
│  Tab content (scrollable)          │
│  bottom padding: 96px              │
│                                    │
├────────────────────────────────────┤
│  Bottom nav (fixed)      82px      │
└────────────────────────────────────┘
```

All four tabs scroll vertically — content height exceeds viewport on all tabs. This is expected and acceptable for an analytics screen (users come here to explore, not for a quick glance).

---

## 16. Animation & Transitions

| Interaction | Animation | Duration | Easing |
|---|---|---|---|
| Tab switch | Fade + slight slide | 200ms | `ease` |
| Period chip switch | Metrics fade out + in, bars re-animate | 300ms | `ease` |
| Bar chart fill on load | Width from 0 | 700ms | `ease-out` |
| Bar chart on period change | Width transition | 500ms | `ease-in-out` |
| Dual bar chart fill | Height from 0 | 700ms | `ease-out`, staggered 100ms per bar |
| Hero amount change | Count animation | 600ms | `ease-out` |
| KPI values on period change | Fade + count | 300ms | `ease-out` |

**Stagger principle for charts:** When bars fill on load, stagger each bar by 80ms (first bar starts at 0ms, second at 80ms, etc.). This creates a wave effect that communicates temporal sequence — earlier months appear before later months.

---

## 17. Accessibility Notes

- All chart bars: `aria-label="[Month] income $X,XXX"` and `aria-label="[Month] spend $X,XXX"`
- Period chips: `role="radio"` group, `aria-checked` per chip
- Risk flags: colour + text — colour never sole indicator
- Progress bars: `aria-valuenow`, `aria-valuemax`, `aria-label`
- Dollar amounts: formatted for screen readers as "two thousand two hundred forty-one dollars"
- Chart containers: `aria-label="[Chart title]"` with summary available in a visually hidden description
