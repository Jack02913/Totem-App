# Totem — Calendar Screen Design Spec
## Visual System & Component Specifications

> **Reference:** All colours use the unified CSS custom properties from `app-mockup.html`. Components inherit from the established design token system — do not hardcode colours.

---

## Screen Anatomy

```
┌─────────────────────────────────────────────┐  ← Status bar (54px, persistent)
│ 9:41                          ···           │
├─────────────────────────────────────────────┤
│  Calendar                              Today│  ← Screen header (46px)
├─────────────────────────────────────────────┤
│     ‹      February 2026      ›             │  ← Month nav bar (44px)
├───────────┬───────────┬────────────────────-┤
│  Spent    │ vs Last   │   Days Left         │  ← Summary strip (52px)
│  $1,157   │ Mo. +8%   │   9                 │
├───────────┴───────────┴─────────────────────┤
│ Sun  Mon  Tue  Wed  Thu  Fri  Sat           │  ← Weekday labels (26px)
├─────────────────────────────────────────────┤
│                                             │
│ [  1 ] [  2 ] [  3 ] [  4 ] [  5 ] ...    │  ← Calendar grid (flex:1 fills remaining)
│ [  8 ] [  9 ] [ 10 ] [ 11 ] [ 12 ] ...    │     4 rows for Feb 2026
│ [ 15 ] [ 16 ] [ 17 ] [ 18 ] [*19*] ...    │     (*today*)
│ [ 22 ] [ 23 ] [ 24 ] [ 25 ] [ 26 ] ...    │
│                                             │
├─────────────────────────────────────────────┤
│  ■ Under   ■ On pace   ■ Over   ■ Payday   │  ← Legend (32px)
├─────────────────────────────────────────────┤
│ Home  Plan  Cal  Accts  Insights            │  ← Bottom nav (82px, persistent)
└─────────────────────────────────────────────┘
```

Total usable screen height (between status bar and nav): 708px
- Screen header: 46px
- Month nav: 44px
- Summary strip: 52px
- Weekday labels: 26px
- Calendar grid: ~508px (flex:1)
- Legend: 32px
= 708px total ✓

---

## Screen Header

```
Component: .screen-header (from established shared style)

Left:   "Calendar" — font-size:22px, font-weight:800, color:var(--text), tracking:-0.5px
Right:  "Today" — font-size:14px, font-weight:600, color:var(--brand)
        Visible only when viewing a month ≠ current month; hidden on current month
        onclick → navigate back to current month, re-centre on today's cell
```

---

## Month Navigation Bar

```
Height: 44px
Padding: 4px 20px 8px

Layout: [← button] [Month + Year title] [→ button]

Navigation buttons (.cal-nav-btn):
  Size:    32×32px
  Shape:   circle (border-radius: 50%)
  Border:  1px solid var(--border)
  BG:      var(--card)
  Icon:    "‹" / "›" at 18px
  Hover:   border-color → var(--brand), color → var(--brand)
  Disabled state (no previous/next month available): opacity:0.3, pointer-events:none

Month title (.cal-month-title):
  Font:    18px, font-weight:800, color:var(--text), tracking:-0.5px
  Center aligned between buttons
```

---

## Month Summary Strip

```
Height:  52px (including border-top and border-bottom)
BG:      var(--card)
Borders: 1px solid var(--border) top and bottom

Layout: 3 equal-width cells, separated by right-border lines

Each cell (.cal-sum-cell):
  Padding:  10px 0
  Align:    text-align: center

  Label:   font-size:10px, font-weight:600, color:var(--text2),
           text-transform:uppercase, letter-spacing:0.5px
  Value:   font-size:16px, font-weight:800, color:var(--text), tracking:-0.5px
           margin-top:2px

Colour overrides for value:
  vs Last Month: red (var(--red)) if positive %, green (var(--green)) if negative %
  Days Left:     amber (var(--amber)) if ≤5 days remaining
```

---

## Weekday Label Row

```
Height:  26px
Padding: 8px 16px 4px
Layout:  CSS grid, 7 equal columns

Labels: Sun, Mon, Tue, Wed, Thu, Fri, Sat
Font:   10px, font-weight:700, color:var(--text3),
        text-transform:uppercase, letter-spacing:0.4px, text-align:center
```

---

## Calendar Grid

```
Layout:  CSS grid, 7 columns, auto rows (height fills remaining space)
         grid-template-columns: repeat(7, 1fr)
         grid-auto-rows: 1fr       ← adapts to 4-row (Feb) or 5-row (most months)
         gap: 3px
Padding: 0 16px 10px
Flex:    flex:1 (fills all remaining space between weekday labels and legend)
         min-height: 0 (required for flex child that contains grid)
```

### Day Cell (.cal-day)

```
Border-radius:  10px
Padding:        6px 5px 5px
Display:        flex, flex-direction:column
Border:         1.5px solid transparent (hover/selected visible)
Overflow:       hidden
Cursor:         pointer
Transition:     background 0.15s

Hover state:    border-color → var(--border)
Selected state: border-color → var(--brand)
```

#### Safety Tint Backgrounds
```
.tint-g   background: rgba(81,207,102,0.09)     ← Green: under daily budget
.tint-a   background: rgba(247,200,75,0.09)     ← Amber: on pace (80–120% of budget)
.tint-r   background: rgba(255,107,107,0.10)    ← Red: over daily budget
.tint-p   background: rgba(139,126,255,0.13)    ← Brand: payday
(none)    background: transparent               ← $0 spend or future day
```

#### Day Cell Child Elements (top to bottom)

**1. Date number (.cal-date-num)**
```
Font:         11px, font-weight:700, color:var(--text2)
Line-height:  1
Margin-bottom: 2px
flex-shrink:  0

TODAY modifier (.is-today .cal-date-num):
  BG: var(--brand)
  Color: #fff
  Border-radius: 50%
  Width/Height: 20×20px
  Display: flex, align-items:center, justify-content:center
  Font: 11px, font-weight:700
```

**2. Payday micro-label (.cal-payday-tag)**
```
Font:     8px, font-weight:700, color:var(--brand)
Transform: uppercase, letter-spacing:0.3px
Line-height: 1
Margin-top: 1px
flex-shrink: 0
Visible only on payday cells
```

**3. Amount (.cal-amount)**
```
Font:        10px, font-weight:700, color:var(--text)
Line-height: 1.2
flex-shrink: 0

Colour modifiers:
  .income  color:var(--brand), font-size:9px  ← net positive days (payday)
  .red     color:var(--red)                   ← optional for deep red tint days
  .amber   color:var(--amber)                 ← optional for amber tint days

Not shown on:
  - $0 days (no amount displayed)
  - Future days (no amount; bills shown via bill flag instead)
```

**4. Category dots (.cal-dots)**
```
Display:    flex, gap:2px, flex-wrap:wrap
Margin-top: auto   ← pushed to bottom of cell
flex-shrink: 0

Each dot (.cal-dot):
  Width/Height: 5×5px
  Border-radius: 50%
  Background:  category colour (var(--red), var(--green), var(--teal), var(--amber), var(--brand))
  flex-shrink: 0
  Max: 3 dots per cell
```

**5. Bill flag (.cal-bill-flag)**
```
Position:    absolute, top:4px, right:4px
Width/Height: 6×6px
Border-radius: 50%

Colours:
  .urgent    background:var(--red)   ← due within 3 days
  (default)  background:var(--amber) ← due 4+ days
```

---

## Blank Cells (month padding)

For months that don't start on Sunday, blank cells fill the leading grid positions:

```
.cal-day.blank
  - No content
  - No background
  - No border
  - pointer-events: none
  - opacity: 0 (invisible but takes up grid space)
```

---

## Legend Bar

```
Height:  32px
Padding: 6px 16px 8px
Layout:  flex, justify-content:center, gap:14px
flex-shrink: 0

Each item (.cal-legend-item):
  Display: flex, align-items:center, gap:5px
  Font:    10px, color:var(--text2)

  Swatch (.cal-legend-swatch):
    Width/Height: 10×10px
    Border-radius: 3px
    Colours: rgba(81,207,102,0.22) / rgba(247,200,75,0.22) / rgba(255,107,107,0.22) / rgba(139,126,255,0.22)

Legend items: "Under" / "On pace" / "Over" / "Payday"
```

---

## Day Detail Bottom Sheet

```
Trigger:     tap any day cell
Mechanism:   standard .sheet-backdrop + .bottom-sheet system (same as Plan screen Add Goal)
Max height:  70% of phone screen height (~590px)
Padding:     8px 20px 40px

Handle:      standard .sheet-handle (32×4px, rgba(255,255,255,0.2))
```

### Sheet Header

```
Date label (.cal-sheet-date-hd):
  Font:    12px, font-weight:600, color:var(--text2)
  Transform: uppercase, letter-spacing:0.5px
  Margin-bottom: 4px
  Example: "THURSDAY, 19 FEB"

Total row (.cal-sheet-total-row):
  Display: flex, justify-content:space-between, align-items:center
  Padding: 0 0 14px
  Border-bottom: 1px solid var(--border)
  Margin-bottom: 12px

  Left label (.cal-sheet-total-lbl):
    Font: 13px, color:var(--text2)
    "Total spent" (past) | "Upcoming" (future) | "Net Day" (payday)

  Right amount (.cal-sheet-total-amt):
    Font: 22px, font-weight:800, color:var(--text), tracking:-0.8px
    .income modifier: color:var(--brand)
```

### Transaction List

```
Reuses existing .list-item pattern (same as Dashboard Upcoming Bills):

  .list-item:
    Padding:    10px 0 (no horizontal padding — sheet has its own padding)
    Border-bottom: 1px solid var(--border)
    :last-child: border-bottom: none

  .list-icon:     36×36px, border-radius:10px, emoji, tinted background
  .list-info:     flex:1
    .list-name:   14px, font-weight:600, color:var(--text)
    .list-sub:    11px, color:var(--text2) — "Category · HH:MM AM/PM"
  .list-right:
    .list-amount: 14px, font-weight:700
                  Spend → color:var(--text), prefix "−"
                  Income → color:var(--brand), prefix "+"

Transaction order:
  1. Income rows (top, with brand-light icon background)
  2. Expenses (descending by amount)
```

### Upcoming Bills Section (future days)

```
Section heading (.cal-upcoming-hd):
  Font:    11px, font-weight:700, color:var(--text2)
  Transform: uppercase, letter-spacing:0.7px
  Margin-bottom: 10px

Each bill row: same .list-item pattern
  Icon background: var(--amber-light)
  Amount colour: var(--amber), prefix "−"
  Sub-label: "Auto-debit · Upcoming"
```

### Empty State

```
For days with no transactions AND no bills:
  Center-aligned text, padding:20px 0
  Font: 13px, color:var(--text2)
  "No transactions on this day."
```

---

## Interaction States & Micro-Animations

### Day Cell Tap
```
1. Selected ring appears: border-color → var(--brand), transition:0.1s
2. Sheet backdrop fades in: opacity 0→0.6, duration:0.35s
3. Sheet slides up: translateY(100%)→translateY(0), cubic-bezier(0.25,0.46,0.45,0.94), 0.35s
```

### Sheet Dismiss
```
Tap backdrop or drag down handle → reverse animation
Sheet slides down → backdrop fades → selected ring removed
```

### Month Transition
```
Switching months:
  Old grid: fadeOut (opacity 1→0, translateX(-20px)), 0.2s
  New grid: fadeIn (opacity 0→1, translateX(20px)→0), 0.25s after old fades
  Summary strip values: cross-fade, 0.3s
```

---

## Contextual Day States — Complete Reference

| State | Date Num | Amount | Dots | Tint | Bill Flag |
|---|---|---|---|---|---|
| Past, no spend | var(--text2) | hidden | none | none | — |
| Past, green | var(--text2) | white $X | 1-3 dots | tint-g | — |
| Past, amber | var(--text2) | white $X | 1-3 dots | tint-a | — |
| Past, red | var(--text2) | white $X | 1-3 dots | tint-r | — |
| Past, payday + spend | var(--text2) | brand "+$X" | 1-3 dots | tint-p | — |
| Today, under | white in brand circle | white $X | 1-3 dots | tint-g | — |
| Today, over | white in brand circle | white $X | 1-3 dots | tint-r | — |
| Future, no bill | var(--text3) | hidden | none | none | — |
| Future, bill | var(--text3) | hidden | none | none | amber dot |
| Future, urgent bill | var(--text3) | hidden | none | none | red dot |
| Future, payday | var(--text3) | hidden | none | tint-p | depends |

---

## Light Theme Adaptations

The safety tints use rgba() opacity on coloured fills, so they naturally adapt to both dark and light themes without needing separate values. The key adaptation needed:

```css
/* Light theme only */
.t-light .cal-day:hover { background: var(--card2); border-color: var(--border); }
.t-light .cal-bill-flag { border: 1px solid var(--bg); } /* prevent bleed on white cells */
```

---

## Web Adaptation (≥1024px)

On desktop, the Calendar screen is not part of the 2-column dashboard layout. It renders as a full-width panel with:
- Larger day cells (the extra width is distributed across 7 columns)
- Day amounts slightly larger (12px instead of 10px)
- 3-column layout for the detail panel (grid + detail always visible side-by-side, no bottom sheet needed)

---

## Accessibility

- Day cells are `role="button"` with `aria-label="February 19, spent $54, 2 categories"`
- Today cell has `aria-current="date"`
- Keyboard navigation: Tab cycles through clickable cells; Enter/Space opens detail
- Safety tint colours all meet 3:1 contrast ratio on the dark navy background (AA for non-text)
- Dot colours are supplementary (size, shape, position convey information without colour alone when all dots are present)
