# Totem — Onboarding Design Specification
## Visual System, Component Rules & Layout

> **Reference file:** See `app-mockup.html` — click "Onboarding Flow" in the sidebar to launch the flow. All `ob-` prefixed CSS classes and JS functions belong to the onboarding system.
> **Platform targets:** iOS (React Native), Android (React Native). Web uses a separate simplified sign-up flow.

---

## 1. Design Principles (Onboarding)

1. **Value before commitment.** The user sees what Totem can do before they're asked to hand over their bank credentials. Trust is earned, not demanded.

2. **Minimal fields, maximum momentum.** Every unnecessary form field is a conversion killer. We ask for: name, email, password (or skip entirely with Apple). We never ask for income, salary, or budget amounts — that's the bank feed's job.

3. **The reveal is the product demo.** Screen 7 (The Reveal) is the single highest-impact moment in acquisition. The user sees their actual money in Totem's UI for the first time. This must feel premium, surprising, and personal.

4. **Progress should always feel forward.** The step dot indicator and back button give users a sense of place and control. Nobody should feel stuck or trapped.

5. **Onboarding ends, but guidance doesn't.** The 8 screens are just the formal handshake. The AI chat bubble continues onboarding progressively, in context, without interrupting the user's exploration.

---

## 2. Overlay Architecture

The onboarding renders as a full-screen overlay inside the phone frame, above all app screens and the bottom nav.

```
CSS:
  position: absolute
  top: 54px  (below status bar — status bar remains visible)
  left: 0; right: 0; bottom: 0
  z-index: 35
  background: var(--bg)

Dismiss animation (.ob-overlay.hidden):
  opacity: 0
  transform: translateY(16px)
  pointer-events: none
  transition: opacity 0.4s, transform 0.4s
```

The onboarding does **not** show the bottom nav or FAB. The status bar (clock + signal icons) remains visible for authenticity.

---

## 3. Screen Transition System

Identical to the `.app-screen` transition system — same easing, same direction convention.

```
.ob-screen:
  position: absolute; top/left/right/bottom: 0
  opacity: 0
  transform: translateX(30px)
  transition: opacity 0.25s ease, transform 0.25s ease

.ob-screen.active:
  opacity: 1; transform: translateX(0)

.ob-screen.exit-left:
  opacity: 0; transform: translateX(-30px)
```

Forward navigation: current screen gets `.exit-left`, new screen gets `.active`.
Back navigation: use the same mechanic (the `.exit-left` naming is directional shorthand, not enforced).

---

## 4. Step Dot Indicator

Shown in the `.ob-header` row on Screens 2–8. Communicates overall progress.

```
.ob-step-dots:
  display: flex; gap: 5px; flex: 1; justify-content: center

Each dot:
  height: 4px; border-radius: 2px; background: var(--text3)
  transition: all 0.3s

Active dot:
  width: 18px; background: var(--brand)

Inactive dots:
  width: 5px
```

6 dots total (Screens 2–8 map to dots 1–6). Screen 1 (Welcome) has no dot indicator — it's the entry, not a step.

---

## 5. Header Row (.ob-header)

Present on Screens 2–8. Contains a back button (left), step dots (centre), and a balancing spacer (right).

```
.ob-header:
  display: flex; align-items: center
  padding: 14px 20px 6px; gap: 12px; flex-shrink: 0

.ob-back (back button):
  width/height: 36px; border-radius: 50%
  background: var(--card); border: 1px solid var(--border)
  font-size: 20px; color: var(--text)
  Content: ‹ (left chevron, not an arrow — thinner, more elegant)

.ob-spacer:
  width: 36px (exact match to .ob-back width, keeps dots centred)
```

Screens 7 and 8 omit the back button (the data has already been fetched — going back has no logical meaning). Only dots are shown, left-aligned.

---

## 6. Body Area (.ob-body)

The scrollable content area of each screen.

```
.ob-body:
  flex: 1
  padding: 10px 24px 24px
  display: flex; flex-direction: column
  overflow-y: auto; scrollbar-width: none
```

Screens with centred content (Screen 5, 6) add `align-items: center; justify-content: center; text-align: center`.

Screens with space-between layout (Screen 1) add `justify-content: space-between`.

---

## 7. Screen 1 — Welcome

```
Layout: space-between, top → hero → features → CTAs

Logo block (top):
  "Totem" — 30px, weight 800, letter-spacing -1.5px, var(--text)
  "." — var(--brand)
  Tagline: 12px, var(--text2)

Hero copy (centre):
  "See your money" 26px, weight 800, letter-spacing -1px, var(--text)
  "clearly" — var(--brand) colour
  Sub: 13px, var(--text2), line-height 1.65

Feature rows (3 items, ob-feature-row):
  Each row: card background, 40px icon box + title + subtitle
  Icons: 💰 brand-light bg · 🤖 teal-light bg · 🔔 amber-light bg

CTAs (bottom):
  Primary: "Get started →" (ob-btn-primary)
  Secondary: "Explore with sample data" (ob-btn-secondary)
  Gap: 10px between buttons
```

**No back button, no step dots** — this is the entry point.

---

## 8. Screen 2 — Account Creation

```
Heading: 22px, weight 800, var(--text)
Sub: 13px, var(--text2)

Sign in with Apple button:
  Background: #fff (always white, regardless of theme)
  Color: #000
  Apple logo SVG + "Continue with Apple" text
  Border-radius: 14px; full-width

Divider: "or" between Apple button and email form

Form fields (3x ob-input):
  First name, Email address, Password
  Gap: 10px
  Focus state: border-color rgba(139,126,255,0.5)

Continue button: ob-btn-primary
Sign in link: 12px, centred, brand colour on "Sign in"

Privacy note:
  margin-top: auto (pushes to bottom regardless of content above)
  font-size: 11px, var(--text3), line-height 1.6
  Links: var(--brand) colour
```

---

## 9. Screen 3 — Bank Selection

```
Heading + sub: same pattern as Screen 2

Bank grid (.ob-bank-grid):
  display: grid; grid-template-columns: 1fr 1fr; gap: 10px

Each bank card (.ob-bank-card):
  padding: 14px; background: var(--card); border: 2px solid var(--border)
  border-radius: 14px; text-align: center

  Emoji: 24px, display block, margin-bottom 6px
  Name: 13px, weight 700, var(--text)

  Selected state:
    border-color: var(--brand); background: var(--brand-light)
    transition: 150ms

Trust badges row (.ob-trust-row):
  3 pills: "🔒 Bank-grade encryption" · "👁 Read-only access" · "✓ CDR accredited"
  flex-wrap: wrap (2+1 wrapping on narrow screens is fine)

CTAs:
  Primary: "Continue →"
  Ghost: "I'll connect later" (12px, brand colour, triggers obExplore())
```

---

## 10. Screen 4 — Pay Cycle

```
Heading + informational sub: same pattern

Option cards (4x .ob-option-card):
  border: 2px solid var(--border) → var(--brand) when selected
  background: var(--card) → var(--brand-light) when selected

  Content:
    Title (14px, weight 600)
    Sub (12px, var(--text2))
    Check circle (.ob-option-check) — right-aligned
      Normal: empty circle, var(--text3) border
      Selected: filled var(--brand), "✓" text inside

Default selection: Fortnightly (pre-selected on screen load)

Tip card:
  Background: var(--card); border: 1px solid var(--border); border-radius: 12px
  Padding: 12px 14px; flex row with 💡 emoji
  Font-size: 12px, var(--text2), line-height 1.55
```

---

## 11. Screen 5 — Bank Auth

```
Layout: centred column, justify-content center, text-align center

Connection animation row:
  [Bank logo box] [animated dots] [Totem T box]

  Bank logo box: 56×56px, border-radius 16px, bank colour tint bg
  Totem box: 56×56px, border-radius 16px, var(--brand-light) bg
             "T" in 22px weight 800 var(--brand)

  Animated dots (.ob-conn-dots):
    3 × 8px circles, var(--brand) fill
    animation: ob-pulse 1.4s infinite, staggered 0.2s each
    Conveys active connection attempt

Heading: 20px, weight 800
Sub: 13px, var(--text2), max-width 280px, line-height 1.65

Trust badges row: same as Screen 3 but smaller set (TLS + OAuth)

Primary CTA: "Open ANZ to authorise →"
Footer note: 11px, var(--text3): "Powered by Basiq · CDR accredited data recipient"
```

---

## 12. Screen 6 — Processing

```
Layout: centred column

Spinner (.ob-spinner):
  48×48px; border: 3px solid var(--border), border-top-color: var(--brand)
  animation: ob-spin 0.9s linear infinite
  margin-bottom: 28px

Heading: 20px, weight 800, var(--text)
Sub: 13px, var(--text2), max-width 280px

Step list card:
  background: var(--card); border: 1px solid var(--border); border-radius: 16px
  padding: 6px 16px

Each step row (.ob-step-row):
  height: ~52px; display flex; align-items center; gap 12px
  border-bottom: 1px solid var(--border)

Step icon states (.ob-step-icon, 28×28px circle):
  Done:    var(--green-light) bg, var(--green) text, "✓"
  Active:  var(--brand-light) bg, var(--brand) text, "◉"
  Pending: rgba(255,255,255,0.05) bg, var(--text3) text, "○"

Step title: 13px, weight 600
Step sub:   11px, colour changes with state (green when done, brand when active, text3 when pending)

Animation sequence (JS-driven, not CSS):
  t=0:      Step 1 done, Step 2 active, counter ticking
  t=ticker: Counter increments to 847
  t+400ms:  Step 2 done (green), Step 3 active
  t+2200ms: Step 3 done, Step 4 active
  t+3400ms: Step 4 done
  t+4000ms: Auto-advance to Screen 7
```

---

## 13. Screen 7 — The Reveal

```
No back button, dots only (5th dot active)

Heading: 22px, weight 800 "Here's what we found 👋"
Sub: 13px, var(--text2)

Reveal hero card (.ob-reveal-card):
  Same gradient as dashboard hero card
  Border: rgba(139,126,255,0.3)
  Border-radius: 20px; padding: 18px

  Label: 11px, var(--brand), uppercase, tracking 1px "SAFE TO SPEND"
  Amount: 38px, weight 800, letter-spacing -2px
  Dollar prefix: 18px, weight 600, var(--text2), vertical-align top
  Sub: 12px, var(--text2) "until payday · after bills & savings"

2-column metric pair (same as dashboard):
  Left:  Top Category (emoji + amount)
  Right: Savings Rate (green colour, directional arrow)

AI flag card (amber variant):
  background: var(--amber-light); border: 1px solid rgba(247,200,75,0.25)
  border-radius: 14px; padding: 12px 14px
  Flex row: emoji + [label (11px, var(--amber), uppercase) + body (12px, var(--text2))]

Decision copy: 14px, weight 600, var(--text) "Does this look right?"

CTAs:
  Primary: "Yes, let's go →"
  Ghost: "Something looks off — I'll fix it"
  Both advance to Screen 8 (corrections happen through the review process)
```

---

## 14. Screen 8 — Transaction Review Onboarding

```
No back button, dots only (6th dot active)

Heading: 22px, weight 800 "One last thing 🎯"
Sub: 13px, var(--text2)

30-transaction progress card:
  background: var(--card); border-radius: 16px; padding: 14px 16px
  Progress label row: "0 of 30 reviewed" (left) + "0%" (right, brand colour)
  Progress bar: 8px height, fill-brand, width 0%
  Sub: 11px, var(--text2)

Review items (3 preview cards, same component as Accounts To Review tab):
  Merchant + date + amount
  Category pill (brand-light background) + confidence badge
  Approve ✓ / Reject ✕ buttons (34×34px circles)
  Separated by 1px border-bottom dividers

CTAs:
  Primary: "Start reviewing →" → obComplete() → Accounts/To Review
  Ghost: "Skip for now, take me to the app" → obComplete() → Dashboard
```

---

## 15. Screen Layout & Proportions

```
Phone: 390×844px
Status bar (above overlay): 54px
Onboarding overlay: 790px tall (844 - 54)
```

None of the 8 onboarding screens require horizontal scrolling. All screens are designed to fit the 790px height with vertical scrolling as a fallback on smaller devices (iPhone SE: 667px total — 54px status = 613px usable; Screens 1, 7, 8 may need slight scroll on SE).

---

## 16. Animation & Transitions

| Interaction | Animation | Duration | Easing |
|---|---|---|---|
| Screen advance | exit-left + slide in from right | 250ms | `ease` |
| Overlay dismiss (explore/complete) | `opacity 0 + translateY(16px)` | 400ms | `ease` |
| Option card select | border-color + background | 150ms | `ease` |
| Bank card select | border-color + background | 150ms | `ease` |
| Connection dots | scale + opacity pulse | 1.4s | `ease`, staggered |
| Processing spinner | continuous rotation | 0.9s | `linear` |
| Step icon transition | instant (JS classname swap) | — | — |
| Progress bar fill (Screen 8) | width from 0% | 700ms on first show | `ease-out` |

---

## 17. Accessibility Notes

- All interactive buttons: minimum 44px tap height (ob-btn padding ensures this)
- Option cards: `role="radio"` group, `aria-checked` per card
- Back button: `aria-label="Go back"`
- Progress bar (Screen 8): `aria-valuenow="0"`, `aria-valuemax="30"`, `aria-label="30 transaction milestone progress"`
- Form inputs: standard `<input>` with `type` attributes; screen reader label via `placeholder` (replace with `<label>` in production)
- Bank auth screen: spinner has `aria-label="Connecting to bank"` + `role="status"`
- Processing step icons: `aria-live="polite"` on the step list container (announces changes)
