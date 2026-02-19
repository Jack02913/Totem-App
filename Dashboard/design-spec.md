# Totem — Dashboard Design Specification
## Visual System, Component Rules & Layout Proportions

> **Reference file:** See `dashboard-mockup.html` for a live rendered example of every component described here.
> **Platform targets:** iOS (React Native), Android (React Native), Web (React). Mobile is primary. Web adapts — see Section 7.

---

## 1. Design Principles

These principles sit behind every decision in this spec. When a new design decision comes up that isn't covered here, run it through these five before deciding.

1. **Decide, don't inform.** Every element on the dashboard should help the user decide what to do — not just tell them what happened. If a metric doesn't change a behaviour, it belongs on a details screen, not the dashboard.

2. **One thing first.** The hierarchy is deliberate. Safe to Spend is always the first prominent number. Everything else earns its position by being more actionable than what came before it.

3. **Colour communicates status, not decoration.** Red = action required. Amber = attention warranted. Green = healthy, no action needed. Brand purple = informational, neutral. Teal = subscription/financial product. Colour is never used purely aesthetically on data — this ensures accessibility and builds user trust (they learn to read colour as signal).

4. **Numbers feel fast.** Hero numbers are large and bold because seeing your number in under 1 second is the product's core promise. Typography weights and letter-spacing are tuned for number readability above everything else.

5. **Progressive complexity.** New users are not overwhelmed. The three-tier system ensures that what fills the first screen is always comprehensible regardless of financial literacy. Complexity is earned by time spent with the product.

---

## 2. Colour System

### Base Palette

| Token | Hex | Usage |
|---|---|---|
| `--bg` | `#0B0F1E` | App background behind all cards |
| `--card` | `#141929` | Standard card background |
| `--card-2` | `#1A2036` | Elevated card (used inside cards for nested metrics) |
| `--border` | `rgba(255,255,255,0.07)` | All card borders, dividers |

**Rationale for dark navy over pure black:** Pure black (#000000) creates harsh contrast that causes eye fatigue for frequent checking. Very dark navy (`#0B0F1E`) reads as "premium dark" while being gentler. It also better supports coloured accents — they appear more vibrant against a warm dark base than against neutral black.

### Semantic Colours

| Token | Hex | Meaning | When to use |
|---|---|---|---|
| `--brand` | `#7C6EFA` | Informational, brand, progress | Progress bars, badges with no urgency, payday indicator, savings trend |
| `--brand-light` | `rgba(124,110,250,0.12)` | Brand background fill | Active state backgrounds, hero card gradient start |
| `--teal` | `#4ECDC4` | Subscriptions & financial products | Subscription card, any recurring financial product context |
| `--teal-light` | `rgba(78,205,196,0.12)` | Teal background fill | Subscription card gradient |
| `--amber` | `#F7C84B` | Warning, at-risk, attention | Payday dot, "At Risk" burn rate, bill due "tomorrow" urgency |
| `--amber-light` | `rgba(247,200,75,0.12)` | Amber background fill | Insight card border glow (warning), at-risk badges |
| `--red` | `#FF6B6B` | Danger, over budget, urgent | "Over Budget" burn rate, "Today" bill urgency, negative safety net |
| `--red-light` | `rgba(255,107,107,0.12)` | Red background fill | Over-budget badge backgrounds |
| `--green` | `#51CF66` | Healthy, on track, positive | "On Track" burn rate, positive projection, safety net healthy |
| `--green-light` | `rgba(81,207,102,0.12)` | Green background fill | On-track badge backgrounds |

**Why these specific colours:** All five semantic colours maintain at minimum 3:1 contrast ratio against `--card` backgrounds (WCAG AA for non-text UI components). The palette is deliberately desaturated relative to standard fintech apps — Totem's emotional tone should feel calm and in-control, not alarming. Only Over Budget (red) should feel urgent.

### Text Colours

| Token | Hex | Usage |
|---|---|---|
| `--text` | `#F0F4FF` | Primary text, numbers, headings |
| `--text-2` | `#8B95B0` | Secondary labels, sub-text, captions |
| `--text-3` | `#3E4660` | Tertiary, muted labels, progress labels, very secondary info |

**Rationale for blue-tinted white:** `#F0F4FF` (slightly blue-shifted white) reads as crisper and cleaner against the dark navy background than pure white `#FFFFFF`. It also sits harmoniously in the brand purple + navy system.

### Gradients

| Usage | Definition |
|---|---|
| Hero card background | `linear-gradient(135deg, #1b1244 0%, #141929 65%)` — deep purple fade |
| Hero progress fill | `linear-gradient(90deg, #7C6EFA, #9C8FFF)` — brand shimmer |
| Subscription card background | `linear-gradient(135deg, #0d1a26 0%, #141929 100%)` — deep teal fade |
| Avatar / initials | `linear-gradient(135deg, #7C6EFA, #9C8FFF)` |

---

## 3. Typography

### Font Stack
```css
font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Display', 'Segoe UI', system-ui, sans-serif;
```

Using the system font stack ensures:
- iOS renders SF Pro (Apple's native, optimised for number display)
- Android renders Roboto/system default
- Web renders Segoe UI on Windows, SF Pro on macOS Safari
- Zero font load time, instant render, native feel on every platform

### Type Scale

| Name | Size | Weight | Letter-spacing | Usage |
|---|---|---|---|---|
| Hero Number | 48px | 800 | -2.5px | Safe to Spend primary value |
| Large Metric | 28–30px | 800 | -1px | Safety Net, Savings Velocity, Subscription count |
| Medium Metric | 22px | 800 | -0.8px | Month Projection, Net So Far, budget amounts |
| Section Heading | 20px | 700 | -0.5px | Tier headings ("Financial Health") |
| Dashboard Greeting | 21px | 700 | -0.5px | "Hey Alex 👋" |
| Body / List Name | 14px | 600 | 0 | Bill names, burn rate categories, list items |
| Supporting Text | 13px | 500 | 0 | Insight card text, hero sub-label, metric sub |
| Caption | 12px | 500 | 0 | Dates, urgency labels, secondary info |
| Label (uppercase) | 11px | 700 | +0.7px | Section titles ("UPCOMING", "BUDGET BURN RATE") |
| Micro Label | 10px | 700 | +0.8–1px | Tier badge, hero metric label, badge text |

**Rationale for tight letter-spacing on large numbers:** Large financial figures (48px) look disproportionately wide at normal letter-spacing. Negative tracking at -2.5px tightens the number to read as a single unit. Test this: "$1,240" at 48px loose spacing looks like a sentence; at -2.5px it reads as one piece of information.

**Rationale for uppercase labels at 10–11px:** Small labels need the extra width that uppercase and tracking provides to remain legible. All-caps + wide tracking is a standard pattern for UI micro-copy labels and creates clear visual hierarchy between the label and the value it describes.

---

## 4. Spacing & Layout System

### Base Unit
**4px** — all padding, margin, and gap values are multiples of 4.

### Standard Spacing Values
| Value | px | Usage |
|---|---|---|
| `xs` | 4px | Icon-to-text gap in badges |
| `sm` | 8px | Payday dot to text, badge padding horizontal |
| `md` | 12px | Gap between metric cards, bill items internal gap |
| `lg` | 14px | Section header top margin |
| `xl` | 16px | Standard card padding, list item padding |
| `2xl` | 20px | Hero card padding |
| `3xl` | 24px | Top-level page padding (sidebar) |
| `4xl` | 32px | Phone container outer padding (desktop) |

### Screen Padding
```
Mobile:  horizontal padding = 20px (left and right)
         bottom padding on scroll area = 96px (clears bottom nav)
```

### Card Border Radius
| Component | Radius | Rationale |
|---|---|---|
| Phone outer frame | 54px | Matches physical iPhone corner radius |
| Phone screen inner | 44px | Standard iPhone screen corner |
| Hero card | 20px | Large, inviting — matches the importance of this card |
| Standard cards | 16px | Consistent for most data cards |
| Nested sub-metrics | 10px | Slightly tighter for nested/smaller components |
| Progress bars | 4px | Subtle, almost rectangular — focuses on data not decoration |
| Badges/pills | 20px | Full pill shape — clearly a label, not a card |

---

## 5. Component Specifications

### 5.1 Dynamic Island Placeholder
```
Width:  120px
Height: 34px
Shape:  border-radius 20px
Colour: #000000
Position: absolute, top 12px, horizontally centred
```
On actual device: hidden behind the real Dynamic Island. On older notched iPhones: position to 12px top works. On Android: omit; use a standard status bar.

### 5.2 Tier Badge + Dot Indicators
```
Tier badge pill:
  font-size: 10px
  font-weight: 700
  padding: 3px 10px
  border-radius: 20px
  background: var(--brand-light)
  color: var(--brand)
  text-transform: uppercase
  letter-spacing: 1px

Dots row:
  display: flex, gap: 5px, centred horizontally

Active dot:
  width: 18px
  height: 4px
  border-radius: 2px
  background: var(--brand)
  transition: width 0.3s, background 0.3s

Inactive dot:
  width: 5px
  height: 4px
  border-radius: 2px
  background: var(--text-3)
```

### 5.3 Payday Pill
```
Layout: horizontal flex, full width
Height: 38px (padding 8px 16px)
Background: var(--card)
Border: 1px solid var(--border)
Border-radius: 24px (full pill)
Margin-bottom: 14px

Internal:
  Dot: 7×7px circle, color var(--amber)
  Text: 13px, color var(--text-2), strong = var(--amber)
  Cycle text: 11px, color var(--text-3), flex-shrink: 0, right-aligned
```

### 5.4 Hero Card (Safe to Spend)
```
Background: linear-gradient(135deg, #1b1244 0%, #141929 65%)
Border: 1px solid rgba(124,110,250,0.3)
Border-radius: 20px
Padding: 20px
Margin-bottom: 12px

Radial glow (::before pseudo):
  position: absolute
  top: -50px, right: -50px
  width/height: 180px
  background: radial-gradient(circle, rgba(124,110,250,0.18) 0%, transparent 70%)
  pointer-events: none

Metric label:
  font-size: 11px, weight 700, color var(--brand)
  text-transform: uppercase, letter-spacing: 1px
  margin-bottom: 4px

Hero amount:
  font-size: 48px, weight 800, color var(--text)
  letter-spacing: -2.5px, line-height: 1

Dollar prefix:
  font-size: 24px, weight 600, color var(--text-2)
  vertical-align: top, margin-top: 10px

Sub-label (below amount):
  font-size: 12px, color var(--text-2)
  margin-bottom: 14px

Progress bar track:
  height: 6px
  background: rgba(255,255,255,0.08)
  border-radius: 4px

Progress fill:
  background: linear-gradient(90deg, var(--brand), #9C8FFF)

Progress labels (beneath bar):
  font-size: 10px, color var(--text-3)
  space-between layout
```

**Proportional reasoning:** The hero card occupies approximately 38% of the visible viewport height above the fold on a standard 844px screen (after status bar and payday pill). This is intentional — the most important single number in the app gets dominant visual weight. Anything larger would feel like a poster; anything smaller would feel underplayed.

### 5.5 Two-Column Metric Pair
```
Display: grid, 2 columns equal width, gap: 10px
Card height: approximately 78px

Each metric card:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 16px
  Padding: 14px 16px

Label: 10px, weight 700, uppercase, letter-spacing 0.7px, color var(--text-2)
Value: 22px, weight 800, letter-spacing -0.8px, color var(--text)
Badge (optional): see Badge spec below
```

### 5.6 AI Insight Card
```
Background: var(--card)
Border: 1px solid rgba(247,200,75,0.25)   // amber glow border — warning state default
Border-radius: 16px
Padding: 14px 16px
Layout: horizontal flex, gap 12px

Icon container:
  width/height: 36px
  border-radius: 10px
  background: var(--amber-light)   // matches border tint
  emoji: 17px

Content:
  Tag: 10px, weight 700, uppercase, letter-spacing 0.8px, color var(--amber)
  Text: 13px, weight 500, color var(--text), line-height 1.5
  CTA link: 12px, weight 600, color var(--brand), margin-top 6px

Border colour variants by insight type:
  Warning (over budget, unusual):  rgba(247,200,75,0.25)  — amber
  Urgent (payday risk, overdraft): rgba(255,107,107,0.25) — red
  Positive (milestone, saving):    rgba(81,207,102,0.25)  — green
  Informational (trend, tip):      rgba(124,110,250,0.25) — brand
```

### 5.7 Upcoming Bills List Card
```
Outer container:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 16px
  overflow: hidden

Each row:
  Height: approximately 60px
  Padding: 12px 16px
  Layout: horizontal flex, gap 12px
  Border-bottom: 1px solid var(--border) (except last row)

Icon:
  width/height: 36px
  border-radius: 10px
  Colour: contextual (Netflix → red, Rent → teal, Spotify → green)
  emoji inside: 16px

Merchant name: 14px, weight 600, color var(--text)
Date/type: 11px, color var(--text-2)

Amount: 14px, weight 700, right-aligned, color var(--text)
Urgency label: 10px, weight 700, uppercase, right-aligned
  - "Today": var(--red)
  - "Tomorrow" / "In 2-3 days": var(--amber)
  - "In 4-7 days" / beyond: var(--text-2)
```

### 5.8 Budget Burn Rate Item
```
Container: same as bills card (card background, bordered)

Each item:
  Padding: 13px 16px
  Border-bottom: 1px solid var(--border)

Top row (flex, space-between):
  Category (emoji + name): 14px, weight 600
  Amount: spent right-aligned (14px, weight 700) / budget (10px, var(--text-2))

Progress bar track:
  height: 6px
  position: relative  // for period marker

Progress fill:
  border-radius: 3px
  Colour: fill-green / fill-amber / fill-red per status

Period marker (white tick):
  position: absolute
  width: 2px, height: 12px (extends 3px above and below the 6px track)
  background: rgba(255,255,255,0.3)
  left: calculated as (days_elapsed / period_length) × 100%
  border-radius: 1px
  z-index: 2
  // This is the key feature — shows where spending SHOULD be at this point

Bottom row (flex, space-between):
  Badge: status badge (see below)
  Percentage: 10px, color var(--text-3)
```

### 5.9 Subscription Load Card
```
Background: linear-gradient(135deg, #0d1a26 0%, var(--card) 100%)
Border: 1px solid rgba(78,205,196,0.2)   // teal glow
Border-radius: 16px
Padding: 18px

Top section:
  Label: 10px, teal, uppercase, tracking 1px
  Count: 30px, weight 800, letter-spacing -1px
  Count unit (" active"): 14px, weight 500, var(--text-2)
  Change badge: right-aligned (see Badge spec)

Sub-metrics grid:
  2-column, gap 10px

Each sub-metric box:
  Background: rgba(255,255,255,0.05)
  Border-radius: 10px
  Padding: 10px 12px
  Label: 10px, var(--text-2)
  Value: 19px, weight 700, letter-spacing -0.5px
    - Monthly: var(--teal)
    - Annualised: var(--amber)  // larger number, slightly alarming by design
```

**Rationale for amber on annualised:** The annualised cost is intentionally shown in amber. Research on mental accounting shows that annual figures activate more deliberate financial thinking than monthly ones. Users seeing "$3,444/year" vs "$287/month" experience greater financial awareness. The amber colour reinforces that this number deserves attention.

### 5.10 Health/Wellness Pair Cards
```
Display: grid, 2 columns equal width, gap 10px

Each card:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 16px
  Padding: 16px

Icon: 20px emoji, margin-bottom 8px
Label: 10px, uppercase, tracking 0.7px, var(--text-2)
Value: 28px, weight 800, letter-spacing -1px, line-height 1
Unit: 11px, var(--text-2), margin-top 3px
Badge: see Badge spec, margin-top 8px
```

### 5.11 Badge / Status Chip
```
Base:
  display: inline-flex
  align-items: center
  gap: 3px
  padding: 2px 7px
  border-radius: 20px
  font-size: 10px
  font-weight: 700

Variants:
  badge-green: background var(--green-light), color var(--green)
  badge-amber: background var(--amber-light), color var(--amber)
  badge-red:   background var(--red-light),   color var(--red)
  badge-brand: background var(--brand-light), color var(--brand)
```

### 5.12 Section Header Row
```
Layout: flex, space-between, align-items: center
Margin: 14px top, 8px bottom

Title: 11px, weight 700, uppercase, letter-spacing 0.8px, color var(--text-2)
Link: 11px, weight 600, color var(--brand), cursor pointer
```

### 5.13 Progress Bar (generic)
```
Track: height 6px, background rgba(255,255,255,0.08), border-radius 4px, overflow hidden
Fill:  height 100%, border-radius 4px
Fill variants: fill-brand, fill-green, fill-amber, fill-red (defined in colour system)
```

---

## 6. Screen Layout & Proportions (Mobile — 390×844px)

### Full Screen Anatomy
```
┌────────────────────────────────────┐  ← 390px wide
│  Dynamic Island       54px        │  status bar
│  [Tier Badge]  [● ○ ○]  10px     │  tier indicator row
├────────────────────────────────────┤
│                                    │
│  TIER PANEL (scrollable)          │  flex: 1 (fills remaining height)
│  bottom padding: 96px              │  clears bottom nav
│                                    │
├────────────────────────────────────┤
│  BOTTOM NAVIGATION   82px fixed   │
└────────────────────────────────────┘
```

### Tier 1 — Content Priority Order & Approximate Heights

| Element | Approx. Height | Priority |
|---|---|---|
| Dashboard header (greeting + avatar) | 56px | P2 — context, not action |
| Payday pill | 38px | P1 — temporal anchor for all numbers below |
| **Hero card (Safe to Spend)** | **~118px** | **P0 — the number** |
| Two-col pair (Month Proj + Net So Far) | 78px | P1 — the so-what |
| Section header | 28px | Structural |
| AI Insight card | ~80–90px | P1 — the action |
| Section header | 28px | Structural |
| Upcoming bills (3 rows) | 180px | P1 — forward-looking |
| Bottom nav (fixed) | 82px | Navigation |

**Total content height:** ~700–710px (fits within 844px viewport minus nav = 762px usable). Tier 1 content is designed to be **fully visible on first render with no scrolling** — this is a hard design constraint. If additional content is added to Tier 1 in future, the tier must still fit without scrolling being required to see the AI Insight.

### Viewport Priority Rule (Tier 1)
```
Safe to Spend hero MUST be visible without scrolling on:
  - iPhone SE (375×667)  ← smallest common iOS target
  - iPhone 14 (390×844)  ← base design target
  - Samsung S22 (360×780) ← base Android target
```

### Above-the-fold Real Estate Allocation
```
First visible viewport (no scroll):
  ~38% → Hero card (Safe to Spend dominant)
  ~10% → Payday pill
  ~10% → Greeting header
  ~10% → Tier bar + status bar
  ~22% → Two-col pair (Month Proj)
  ~10% → AI Insight (top portion visible — scroll prompt)
```

The last 10% of AI Insight being partially visible is intentional — it acts as a scroll affordance, signalling that there's more below.

---

## 7. Web (Desktop) Adaptation

On desktop (viewport ≥ 1024px wide), the tier system changes behaviour. Swiping between tiers is a mobile metaphor that doesn't translate to mouse-driven interaction.

### Web Dashboard Layout
```
┌─────────────────────────────────────────────────────────┐
│  Totem    [Home] [Goals] [Forecast] [Net Worth] [Cal]   │  ← top nav
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────┐  ┌───────────────────────────┐  │
│  │  TIER 1 — OVERVIEW │  │  TIER 2 — HEALTH          │  │
│  │  Safe to Spend     │  │  Budget Burn Rate         │  │  ← side-by-side columns
│  │  Month Projection  │  │  Subscription Load        │  │    on wide screens
│  │  AI Insight        │  │  Safety Net / Velocity    │  │
│  │  Upcoming Bills    │  │                           │  │
│  └────────────────────┘  └───────────────────────────┘  │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  TIER 3 — TRENDS (full width, below)            │    │  ← full width below
│  │  Lifestyle Creep  │  Category % Income  │ Trend │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

- Tier 1 and Tier 2 display as equal-width columns
- Tier 3 spans full width below both (its content is naturally wider — charts benefit from horizontal space)
- No tier switching animation — all tiers visible simultaneously
- Desktop does not show the tier dots or tier badge pill
- Cards maintain the same internal proportions; the container width simply expands

### Responsive Breakpoints
```
< 640px  (mobile):   single column, tier swipe system
640–1023px (tablet): single column, tier swipe, wider padding
≥ 1024px (desktop):  2-column layout, all tiers visible
≥ 1440px (wide):     max-width: 1200px, centred
```

---

## 8. Animation & Transitions

| Interaction | Animation | Duration | Easing |
|---|---|---|---|
| Tier slide | `transform: translateX()` | 400ms | `cubic-bezier(0.25, 0.46, 0.45, 0.94)` |
| Tier dot expand | `width` change | 300ms | `ease` |
| Number update (new transaction) | Count-up animation | 600ms | `ease-out` |
| Card press/tap | Scale to 0.98 | 100ms | `ease` |
| Badge appear | Fade + scale from 0.9 | 200ms | `ease-out` |
| Progress bar fill (on load) | Width from 0 | 800ms | `ease-out` |

**Principle:** Animate the number changes. When Safe to Spend updates because a new transaction came in, the hero number should count down (or up) visually. This makes the connection between "spent money" and "less to spend" visceral and immediate.

---

## 9. Bottom Navigation

```
Height: 82px (includes home indicator area on iPhone)
Background: rgba(11,15,30,0.92) + backdrop-filter: blur(20px)
Border-top: 1px solid var(--border)
Padding-top: 12px (content sits above home indicator)

5 tabs: Home, Goals, Forecast, Net Worth, Calendar

Each tab:
  icon: 22×22px SVG
  label: 10px, weight 500
  Active: icon + label = var(--brand)
  Inactive: icon + label = var(--text-3)
  Tap area: 44×44px minimum (iOS HIG accessibility requirement)
```

---

## 10. Empty States

Each feature has an empty/building state for new users.

| Feature | Empty State Display |
|---|---|
| Safe to Spend | "Connect a bank account to see your available spending" |
| Budget Burn Rate | "Set spending budgets to track your progress" [+ Set budgets CTA] |
| AI Insight | Hidden — do not show an empty insight card |
| Upcoming Bills | "We haven't detected any recurring payments yet. Connect your bank to start." |
| Safety Net | Shows if 30+ days of data. Otherwise: "Building data — check back in 30 days" |
| Savings Velocity | Shows after first complete pay period |
| Lifestyle Creep | Progress bar ("X of 3 months complete") — see feature-list.md |
| Category % of Income | Shows from day 1 (as long as transactions exist) |

---

## 11. Accessibility Notes

- All interactive elements: minimum 44×44px tap target (iOS HIG)
- Colour is never the only indicator of status — badges always include text labels ("Over Budget", not just a red bar)
- Font sizes below 12px are decorative only — no critical information below 12px
- Contrast ratios: all primary text (`--text` on `--card`) achieves 7:1 — WCAG AAA
- Progress bars include `aria-valuenow`, `aria-valuemin`, `aria-valuemax`, `aria-label` attributes
- Hero number should be readable by screen reader as: "Safe to spend, $312"
