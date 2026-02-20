# Totem — Accounts Design Specification
## Visual System, Component Rules & Layout Proportions

> **Reference file:** See `app-mockup.html` (all `acct-` prefixed IDs and functions) for a live rendered example of every component described here.
> **Platform targets:** iOS (React Native), Android (React Native), Web (React). Mobile is primary.

---

## 1. Design Principles (Accounts Screen)

1. **Trust through transparency.** The Accounts screen is where users see their raw financial reality. Design must communicate accuracy and liveness — users must feel the data is real-time, not stale.

2. **Make reviewing effortless.** The To Review tab is a critical data-quality task. Friction here = worse AI recommendations everywhere else. Approve in one tap, correct in two. Animations reward completion.

3. **Rules feel like control.** The Rules tab should feel like a settings panel the user owns — clear, inspectable, editable. Not a black box.

4. **Progress motivates.** The 30-transaction gate is a gamification mechanic. The progress bar must feel rewarding at each increment — smooth animation, visible change, clear proximity to the milestone.

---

## 2. Tab Navigation (Accounts | To Review | Rules | Subs)

Same pattern as Plan screen (`switchPlanTab`) and Insights screen (`switchInsTab`).

```
Tab bar:
  Position: below the screen header, sticky (does not scroll with content)
  Height: 44px
  Background: var(--card)
  Border-bottom: 1px solid var(--border)

Each tab:
  flex: 1 (equal width)
  font-size: 14px
  font-weight: 600
  text-align: center
  padding: 12px 0
  color: var(--text-2)  // inactive

Active tab:
  color: var(--brand)
  border-bottom: 2px solid var(--brand)  // bottom underline indicator
  margin-bottom: -1px  // overlaps tab bar border
```

**Tab switching:** `switchAcctTab(tab)` sets active tab class, shows/hides `.acct-tab-*` content divs, and calls `updateAcctHeader(tab)` to update the screen header.

---

## 3. Screen Header

The header changes content based on the active tab. Controlled by `updateAcctHeader(tab)`.

```
Base header:
  Height: 56px
  Padding: 0 20px
  Display: flex, align-items: center, justify-content: space-between
  Border-bottom: 1px solid var(--border)

Accounts tab header:
  Left: "Accounts" (20px, weight 700, var(--text))
  Right: Live feed indicator dot (see Section 6.1) + "+ Add" button

To Review tab header:
  Left: "To Review" (20px, weight 700)
  Right: "24/30" progress label (13px, var(--text-2))

Rules tab header:
  Left: "Rules" (20px, weight 700)
  Right: "+ Add Rule" button (13px, var(--brand), weight 600)
```

---

## 4. Accounts Tab

### 4.1 Net Position Hero Card

```
Container:
  Background: linear-gradient(135deg, #1b1244 0%, #141929 65%)  // same as dashboard hero
  Border: 1px solid rgba(124,110,250,0.25)
  Border-radius: 20px
  Padding: 20px
  Margin: 16px 20px 8px

Label row:
  font-size: 11px, weight 700, color var(--brand)
  text-transform: uppercase, letter-spacing: 1px
  Content: "NET POSITION"

Hero amount:
  font-size: 40px, weight 800, letter-spacing: -2px, color var(--text)
  Prefix ($): font-size 20px, weight 600, var(--text-2), vertical-align top, margin-top 8px
  Value: positive → var(--brand); negative → var(--red)

Sub-line:
  font-size: 12px, color var(--text-2)
  Content: "$14,297 assets · $843 debt"
  Margin-top: 6px
```

### 4.2 Live Feed Indicator

```
Dot component:
  width/height: 8px
  border-radius: 50%
  display: inline-block
  margin-right: 6px

States:
  Active (synced < 4h):
    background: var(--green)
    animation: pulse-green 2s infinite
    // @keyframes pulse-green: 0%/100% opacity 1, 50% opacity 0.5 + scale(1.3)

  Warning (synced 4-24h):
    background: var(--amber)
    animation: none

  Error (synced > 24h or disconnected):
    background: var(--red)
    animation: none

Tooltip (tap the dot):
  Small popover: "Live · Last updated 3 min ago"
  font-size: 12px, background var(--card-2), border-radius 8px, padding 6px 10px
```

### 4.3 Account Row

```
Outer container:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 16px
  Margin: 0 20px 8px
  overflow: hidden

Each account row:
  Padding: 14px 16px
  Display: flex, align-items: center, gap: 12px
  Border-bottom: 1px solid var(--border)  // except last row

Account icon:
  width/height: 40px
  border-radius: 12px
  Background: institution colour (ANZ → red tint; Westpac → red-orange; CBA → yellow; NAB → dark)
  // Fallback: var(--brand-light) with institution initials
  font-size: 14px, weight 700, text-align center

Account info (flex: 1):
  Name: 14px, weight 600, var(--text)
  Type label: 11px, var(--text-2)  // "Savings · 3.75% p.a." for savings accounts

Balance (right-aligned):
  Amount: 15px, weight 700, var(--text)
  Negative (credit card): var(--red)
  Live dot: 6px dot inline before balance (green if account synced, amber/red if stale)

Tap area: full row → opens account detail bottom sheet (openAcctDetail)
```

---

## 5. Account Detail Bottom Sheet

Opened by `openAcctDetail(accountId)`. Standard bottom sheet pattern: `.acct-acct-sheet` + `.sheet-backdrop#acct-acct-backdrop`.

```
Sheet:
  Position: fixed, bottom 0, left 0, right 0
  Max-height: 70vh
  Background: var(--card)
  Border-radius: 24px 24px 0 0
  Padding: 0 0 32px
  Transform: translateY(100%) → translateY(0) on open
  Transition: transform 0.35s cubic-bezier(0.25, 0.46, 0.45, 0.94)

Drag handle:
  width: 36px, height: 4px
  Background: var(--text-3)
  Border-radius: 2px
  Margin: 12px auto 8px
  display: block

Sheet header:
  Padding: 12px 20px
  Institution name: 16px, weight 700
  Account name + masked number: 13px, var(--text-2)
  Balance: 28px, weight 800, letter-spacing: -1px (large, prominent)
  Interest rate badge (savings only): badge-brand, "3.75% p.a."

Section label:
  font-size: 11px, weight 700, uppercase, letter-spacing 0.8px
  color: var(--text-2), padding: 12px 20px 8px

Transaction list:
  Each row (padding 12px 20px, border-bottom var(--border)):
    Left: Category emoji (20px) + merchant name (14px, weight 600)
    Sub: Date (11px, var(--text-2))
    Right: Amount (14px, weight 700)
      - Debit: positive display, var(--red)
      - Credit/income: show with +, var(--green)

Footer link:
  "See all transactions →"
  font-size: 13px, color var(--brand), weight 600
  text-align: center, padding: 16px
```

---

## 6. To Review Tab

### 6.1 Progress Bar (30-Transaction Gate)

```
Container:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 16px
  Padding: 16px
  Margin: 16px 20px 8px

Progress label row (flex, space-between):
  Left: "24 of 30 reviewed" (13px, weight 600, var(--text))
  Right: "80%" (13px, var(--brand), weight 700)

Bar track:
  height: 8px  // slightly taller than standard — this is a milestone bar
  background: rgba(255,255,255,0.08)
  border-radius: 4px
  margin: 10px 0

Bar fill:
  transition: width 0.4s ease-out
  border-radius: 4px
  < 30/30: background var(--brand)
  = 30/30: background var(--green) + brief flash animation on milestone

Sub-label:
  font-size: 12px, color var(--text-2)
  Content: "Review transactions to unlock AI insights and goal suggestions"

Milestone state (30/30):
  Bar fill: var(--green), full width
  Replace sub-label with: "✓ AI features unlocked" (12px, var(--green), weight 600)
  Brief scale + colour flash on the milestone moment (keyframe: scale 1 → 1.02 → 1, 400ms)
```

### 6.2 Review Item Card

```
Outer card:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 16px
  Margin: 0 20px 8px
  overflow: hidden

Collapse animation (on approve or correct):
  max-height: 200px → 0
  opacity: 1 → 0
  padding: card padding → 0
  transition: max-height 0.3s ease-out, opacity 0.2s ease-out
  // After transition: element removed from DOM or display:none

Item content row (padding 14px 16px):
  Merchant name: 14px, weight 600, var(--text)
  Date: 11px, var(--text-2)
  Amount: 15px, weight 700, right-aligned, var(--text)

Second row (flex, space-between):
  Left: AI category pill + confidence badge
  Right: Action buttons (✓ and ✗)
```

### 6.3 Category Pill + Confidence Badge

```
Category pill:
  display: inline-flex, align-items: center, gap: 4px
  padding: 4px 10px
  border-radius: 20px
  background: var(--brand-light)
  color: var(--brand)
  font-size: 12px, weight 600
  Content: emoji + category name  // e.g. "🍔 Eating Out"

Confidence badge (immediately right of pill):
  Base: same as dashboard Badge / Status Chip spec
  "High":   badge-green
  "Medium": badge-amber
  "Low":    badge-red
  font-size: 10px, weight 700
```

### 6.4 Approve / Reject Buttons

```
Button container:
  display: flex, gap: 8px

Approve (✓) button:
  width/height: 36px
  border-radius: 50% (circular)
  background: var(--green-light)
  color: var(--green)
  font-size: 16px (checkmark icon)
  border: 1px solid rgba(81,207,102,0.3)
  tap: calls approveTransaction(id)

Reject (✗) button:
  width/height: 36px
  border-radius: 50%
  background: var(--red-light)
  color: var(--red)
  font-size: 14px (×)
  border: 1px solid rgba(255,107,107,0.3)
  tap: calls openCatSheet(id)  // opens category picker

Both buttons:
  active state: scale(0.92) on press (100ms)
  Minimum tap area: 44×44px (extend hit area with padding)
```

### 6.5 Empty State (All Reviewed)

```
Container: centred, padding 40px 20px

Icon: large checkmark in circle (40px, var(--green))
Heading: "All caught up!" (18px, weight 700, var(--text))
Sub: "Nice work. We'll add more as new transactions come in." (13px, var(--text-2))

Progress note below:
  Same 30-transaction progress bar (always shown regardless of empty state)
```

---

## 7. Category Picker Bottom Sheet

Opened by `openCatSheet(transactionId)`. Sheet ID: `acct-cat-sheet`. Backdrop: `acct-cat-backdrop`.

```
Sheet (same base as account detail sheet):
  Max-height: 55vh
  Drag handle: same spec

Header:
  "Change Category" (16px, weight 700, var(--text))
  Close × button (right-aligned, 20px, var(--text-2))

Grid:
  display: grid
  grid-template-columns: repeat(3, 1fr)
  gap: 10px
  padding: 16px 20px

Each category cell:
  Background: var(--card-2)
  Border: 1px solid var(--border)
  Border-radius: 12px
  Padding: 14px 8px
  Text-align: center
  cursor: pointer

  Emoji: 22px, display block, margin-bottom 6px
  Name: 11px, weight 600, var(--text-2), line-height 1.3
  // Two-line names (e.g. "Personal\nCare") use line-break

  Hover/Active state:
    background: var(--brand-light)
    border-color: rgba(124,110,250,0.4)
    Name colour: var(--brand)
    transition: all 0.15s ease

  On tap: calls pickCategory(categoryId) → closes sheet → triggers correct flow
```

---

## 8. Rules Tab

### 8.1 Rules List

```
Container:
  Background: var(--card)
  Border: 1px solid var(--border)
  Border-radius: 16px
  Margin: 16px 20px 8px
  overflow: hidden

Each rule row:
  Padding: 14px 16px
  Display: flex, align-items: center, gap: 12px
  Border-bottom: 1px solid var(--border)  // except last

Rule icon:
  width/height: 36px, border-radius: 10px
  Background: category colour (light tint)
  Emoji: 16px

Rule info (flex: 1):
  Merchant name: 14px, weight 600, var(--text)
  Category name: 11px, var(--text-2)
  Source badge: immediately right of merchant name (see 8.2)

Right side:
  ON toggle (see 8.3)
  Trash icon (see 8.4)
```

### 8.2 Source Badge

```
"You" badge:
  display: inline-flex
  padding: 2px 7px
  border-radius: 20px
  background: var(--brand-light)
  color: var(--brand)
  font-size: 10px, weight 700

"✨ AI" badge:
  Same dimensions
  background: rgba(78,205,196,0.12)  // teal light
  color: var(--teal)
  Content: "✨ AI"
```

### 8.3 ON Toggle

```
Toggle track:
  width: 38px, height: 22px
  border-radius: 11px
  background: var(--green) when ON, var(--text-3) when OFF
  transition: background 0.2s ease
  cursor: pointer

Toggle thumb:
  width: 18px, height: 18px
  border-radius: 50%
  background: white
  position: absolute
  top: 2px
  left: 2px (OFF) → left: 18px (ON)
  transition: left 0.2s ease
  box-shadow: 0 1px 3px rgba(0,0,0,0.3)

Tap: calls toggleRule(ruleId) → updates is_active, animates thumb
// In mockup: visual-only, no backend call
```

### 8.4 Trash / Delete Icon

```
Icon: 🗑 or SVG trash
Size: 16px
Color: var(--text-3)
Padding: 8px (enlarged tap area)
Active state: color var(--red) on press
Tap: calls deleteRule(ruleId) → confirmation alert → removes row with fade-out
// In mockup: decorative only

IMPORTANT: Both the toggle and the trash icon call event.stopPropagation() to
prevent the parent rule row click from triggering the edit sheet.
```

### 8.5 Rule Edit Bottom Sheet (`acct-rule-edit-sheet`)

```
Trigger: tap anywhere on a rule row (not the toggle or trash)
Height: max-height 74%; overflow-y: auto

Header context row:
  Icon (list-icon, 40×40px) + merchant name (16px, weight 800) + current category (12px, var(--text2))
  This row is read-only — shows what you're editing

Category picker grid (.acct-cat-grid inside #rule-cat-grid):
  3 columns, 11 category options
  Each option: emoji (20px) + category name (11px)
  Selected state: border-color var(--brand), background var(--brand-light)
  onclick: pickRuleCat(el) — toggles .selected on clicked option

Note: The Add Rule sheet (acct-add-rule-sheet) uses id="add-rule-cat-grid"
and pickAddRuleCat() to avoid ID collision with the edit sheet.

Action buttons (full-width, stacked):
  "Cancel" — closeRuleEdit()
  "Save Rule" (primary) — saveRuleEdit()
    → Updates row icon, category label, data-cat attribute
    → Downgrades "✨ AI" badge to "You" if rule was AI-sourced
    → Toast: "Rule updated — future transactions recategorised"
```

---

## 9. Subscriptions Tab (Subs)

### 9.1 Subs Pane

```
Container: .tab-pane#acct-pane-subs
Padding: 16px 0

Section heading: "Your Subscriptions" (section-hd pattern)
List: .list-card (card container with border-radius 16px)
```

### 9.2 Subscription Row

```
Each row: .sub-row
  padding: 14px 16px (scoped to .list-card .sub-row to avoid bleeding)
  display: flex; align-items: center; gap: 12px
  border-bottom: 1px solid var(--border)  // except last

Left icon: .list-icon
  width/height: 36px; border-radius: 10px; flex-shrink: 0
  background: service-brand tint (rgba at 0.12 opacity)
  font-size: 15px (emoji)

Centre (flex: 1):
  Service name: 14px, weight 600, var(--text)
  Category tag: 11px, var(--text2)

Right:
  Amount: 13px, weight 600, var(--text)
  Frequency: 10px, var(--text3) (e.g. "/ mo")
```

### 9.3 Icon Colour Mapping

| Service | Background | Emoji |
|---|---|---|
| Netflix | `rgba(229,9,20,0.12)` | 🎬 |
| Spotify | `rgba(30,215,96,0.12)` | 🎵 |
| Apple TV+ | `rgba(0,125,250,0.12)` | 📺 |
| YouTube Premium | `rgba(255,0,0,0.12)` | 📹 |
| Disney+ | `rgba(17,60,207,0.12)` | 🏰 |
| iCloud+ | `rgba(0,125,250,0.12)` | ☁️ |
| Amazon Prime | `rgba(255,153,0,0.12)` | 🛍️ |
| ChatGPT Plus | `rgba(16,163,127,0.12)` | 🤖 |
| F45 Training | `rgba(124,110,250,0.12)` | 🏋️ |
| Adobe CC | `rgba(255,0,0,0.12)` | 🎨 |
| Headspace | `rgba(255,161,0,0.12)` | 🧘 |

The `.list-icon` pattern (rounded square with brand-tinted background) is the same component used in the To Review tab and the Insights Top Merchants section. Consistent across all list contexts.

---

## 10. Screen Layout & Proportions (Mobile — 390×844px)

```
┌────────────────────────────────────┐
│  Status bar              54px      │
├────────────────────────────────────┤
│  Screen header           56px      │  title + right action
├────────────────────────────────────┤
│  Tab bar                 44px      │  Accounts | To Review | Rules | Subs
├────────────────────────────────────┤
│                                    │
│  Tab content (scrollable)          │  flex: 1
│  bottom padding: 96px              │  clears bottom nav + FAB
│                                    │
├────────────────────────────────────┤
│  Bottom nav (fixed)      82px      │
└────────────────────────────────────┘
```

### Accounts Tab Content Height Budget (approximate)
| Element | Height |
|---|---|
| Net Position hero card | ~100px |
| Account rows (3 accounts) | 3 × ~64px = 192px |
| "Add account" row | ~48px |
| **Total** | ~340px (no scroll required for 3 accounts) |

---

## 11. Animation & Transitions

| Interaction | Animation | Duration | Easing |
|---|---|---|---|
| Tab switch | Content fade + slide | 200ms | `ease` |
| Review item collapse (approve/correct) | `max-height` + `opacity` | 300ms / 200ms | `ease-out` |
| Progress bar fill increment | `width` | 400ms | `ease-out` |
| Milestone reached (30/30) | Scale pulse + colour change | 400ms | `ease-in-out` |
| Bottom sheet open | `translateY` | 350ms | `cubic-bezier(0.25, 0.46, 0.45, 0.94)` |
| Bottom sheet close | `translateY` | 300ms | `ease-in` |
| Account row tap feedback | Scale 0.98 | 100ms | `ease` |
| Toggle switch | `left` + `background` | 200ms | `ease` |
| Rule delete fade | `opacity` + `max-height` | 250ms | `ease-out` |

---

## 12. Accessibility Notes

- All interactive rows (account rows, review items, rule rows): minimum 44×44px tap target
- Approve/reject buttons: 44×44px tap area (circular button + padding)
- Colour never sole indicator: confidence badges include text ("High", "Medium", "Low") not just colour
- Toggle state: includes `aria-checked` attribute for screen readers
- Progress bar: `aria-valuenow`, `aria-valuemax`, `aria-label="30-transaction milestone progress"`
- Bottom sheet: focus trap when open; `aria-modal="true"`, close on Escape key
