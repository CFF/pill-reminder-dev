---
name: Posology
colors:
  bg: '#f7ede0'
  bg2: '#e8ddd1'
  card: '#ffffff'
  card2: '#faf7f3'
  sep: 'rgba(0,0,0,0.065)'
  sep-strong: 'rgba(0,0,0,0.13)'
  t1: '#18181b'
  t2: '#5c5c66'
  t3: 'rgba(0,0,0,0.28)'
  acc: '#d45500'
  acc-soft: 'rgba(212,85,0,0.10)'
  acc-glow: 'rgba(212,85,0,0.22)'
  green: '#1eb854'
  blue: '#0066e8'
  red: '#e5342c'
  gold: '#a34d00'
  gold-bg: 'rgba(163,77,0,0.09)'
  violet: '#6d28d9'
  violet-bg: 'rgba(109,40,217,0.09)'
colors-dark:
  bg: '#161618'
  bg2: '#232325'
  card: '#252528'
  card2: '#2e2e32'
  sep: 'rgba(255,255,255,0.08)'
  sep-strong: 'rgba(255,255,255,0.15)'
  t1: '#f0f0f5'
  t2: '#a0a0ab'
  t3: 'rgba(255,255,255,0.26)'
  acc: '#ff6b1a'
  acc-soft: 'rgba(255,107,26,0.14)'
  acc-glow: 'rgba(255,107,26,0.28)'
  green: '#2ed15f'
  blue: '#3b9eff'
  red: '#ff4d45'
  gold: '#f5a623'
  gold-bg: 'rgba(245,166,35,0.12)'
  violet: '#a855f7'
  violet-bg: 'rgba(168,85,247,0.13)'
shadows:
  sm: '0 1px 2px rgba(0,0,0,0.07), 0 1px 3px rgba(0,0,0,0.05)'
  md: '0 2px 5px rgba(0,0,0,0.08), 0 1px 2px rgba(0,0,0,0.05)'
  lg: '0 4px 14px rgba(0,0,0,0.10), 0 1px 3px rgba(0,0,0,0.06)'
typography:
  display:
    fontFamily: 'DM Serif Display'
    fallback: 'Georgia, serif'
    className: serif
  body:
    fontFamily: 'DM Sans'
    fallback: 'system-ui, sans-serif'
  weights:
    regular: 400
    medium: 500
    semibold: 600
    bold: 700
  sizes:
    xs: 11
    sm: 12
    base: 13
    md: 14
    lg: 15
    xl: 16
    xxl: 17
    display-sm: 20
    display-md: 26
    display-lg: 34
    splash: 52
spacing:
  base: 4
  xs: 4
  sm: 6
  md: 8
  lg: 10
  xl: 12
  2xl: 14
  3xl: 16
  4xl: 20
  5xl: 24
  6xl: 28
  7xl: 32
radius:
  xs: 6
  sm: 8
  md: 12
  lg: 16
  xl: 18
  2xl: 20
  3xl: 22
  full: 100
---

# Posology — Design System

The visual system: identity, color, type, spacing, component conventions, and known design debt. For build details see **SPECS.md**. For behavioral rules see **CLAUDE.md**.

## Identity

iOS-native feel, warm and calm. This is a daily health tool used by patients, often anxiously, sometimes one-handed. The design serves the task: read fast, tap confidently, never feel clinical or cold.

Core signals:
- **Orange (`--acc`) = in progress.** Green (`--green`) = fully complete. This pairing carries the app's primary status language.
- Warm paper background (`--bg`), grouped list cards (`--card`), inset separators, generous tap targets.
- Warmth comes from the accent and the serif display type, not from piling on more cream surfaces.

## Color

Auto light/dark via `prefers-color-scheme`. Tokens are defined in the YAML frontmatter above. Rules below govern intent and constraints.

### Semantic usage

| Token | Purpose | Constraints |
|---|---|---|
| `--bg` | App background | Warm paper. Do not add more tinted surfaces — carry warmth through `--acc` and type instead. |
| `--bg2` | Recessed surface, assistant bubbles | One step darker than `--bg`. Use for inset areas, not for cards. |
| `--card` | Cards, sheet headers, input bars | Pure white in light mode. Primary container surface. |
| `--card2` | Secondary card surface | Slightly warm. Use for nested rows inside a card, not as a standalone card. |
| `--t1` | Primary text | Headings, pill names, any text the user must read. |
| `--t2` | Secondary text | Dose, timing, subtitles, labels. |
| `--t3` | Decorative only | **Never use on text that must be read.** Contrast is ~1.9:1, well under 4.5:1. Reserved for ornamental dividers and ghosted placeholders. |
| `--acc` | Accent, in-progress state, primary actions | In-progress sessions, CTA buttons, active checkmarks, pill shape icons. Do not use for success or error states. |
| `--acc-soft` | Soft accent fill | Pill icon background, warning chips, soft highlights behind accent elements. |
| `--acc-glow` | Accent glow | Header gradient overlays only. Do not use on cards. |
| `--green` | Fully complete | Session completion, dose count chips, "last taken" labels. |
| `--blue` | New / informational | New tip badges, navigation back links, interval-blocked "Next at" chips. |
| `--red` | Destructive | Delete actions, error states only. |
| `--gold` / `--gold-bg` | Inventory warning | Low-stock badge text and background. |
| `--violet` / `--violet-bg` | Renewable badge | Renewable inventory indicator. |
| `--sep` | Default separator | Between rows inside a card. |
| `--sep-strong` | Strong separator | Disclaimer borders, tip card outlines, structural dividers. |

## Typography

### Typefaces

- **DM Serif Display** (`.serif` class, falls back to Georgia). Used for screen titles — History, today's date, app name on splash. The warm, human voice of the app.
- **DM Sans** (body, falls back to system-ui). Used for everything else — pill names, labels, buttons, metadata. Not Inter, not system font, despite older docs.
- Both loaded from Google Fonts.

### Usage rules

- Use DM Serif Display only for top-level page titles (h1 equivalent). Never for labels, buttons, or metadata.
- Use DM Sans for all body copy, form elements, chips, and captions.
- Font sizes follow a strict scale (see YAML). Do not use values outside the defined scale.
- Weights in use: 400 (body), 500 (medium labels), 600 (semibold section headers, pill names), 700 (bold chips, warning text).
- Letter-spacing: -0.01em on large titles, 0.04–0.06em on uppercase labels, default (0) on body.
- Uppercase is reserved for section group headers (e.g., "MORNING", "AS NEEDED") — never use it for primary content.

## Spacing

Spacing is built on a 4px base unit. Common values and their roles:

| Token | Value | Common use |
|---|---|---|
| `xs` | 4px | Icon-to-text gap, tight inline gaps |
| `sm` | 6px | Icon-to-label gap in section headers |
| `md` | 8px | — |
| `lg` | 10px | Between pill cards in a session |
| `xl` | 12px | — |
| `2xl` | 14px | Section header bottom margin |
| `3xl` | 16px | Horizontal page padding, card inner padding |
| `4xl` | 20px | Header vertical padding |
| `5xl` | 24px | — |
| `6xl` | 28px | Session block bottom margin |
| `7xl` | 32px | GroupCard bottom margin |

All scroll containers: `paddingBottom: calc(90px + env(safe-area-inset-bottom))` to clear the tab bar.

## Shadows

Three levels. Always use the CSS variable, never hardcode.

- `--shadow-sm` — subtle lift. Use on chips, badges, small elements.
- `--shadow-md` — pill cards, sheet headers.
- `--shadow-lg` — bottom sheets, modals, anything floating above the content layer.

Dark mode shadows are heavier (higher alpha) to compensate for reduced contrast.

## Component conventions

- **Border radius:** Pill cards `18px` (`--radius-xl`). Group cards `16px` (`--radius-lg`). Chips and badges `6px` (`--radius-xs`). Full pill shape: `100px`.
- **Tap targets:** 44px minimum on all interactive elements (iOS HIG).
- **Safe area:** `env(safe-area-inset-*)` on all edges. Never hardcode margins near the screen edges.
- **Primary action:** Full-width accent button (`background: --acc`, white text, `fontWeight: 700`). Cancel: full-width text link below it, never side by side.
- **Card shadow + border:** `--shadow-md` + `0.5px solid var(--sep)` on interactive cards.

---

## Components

### SplashScreen

Shown on cold start while data loads. Hidden by `window.hideSplash()` once `loadData()` resolves.

| State | Visual |
|---|---|
| Loading | Logo wordmark in DM Serif, capsule icon, shimmer progress bar (infinite animation) |
| Fading out | 0.35s opacity fade triggered by `hideSplash()`, then `display:none` |
| Gone | Element removed from flow with `.gone` class |

**Constraints:** Never show a spinner or loading text. The shimmer bar is the only loading affordance. Do not add more visual elements to this screen.

---

### TabBar

Fixed to the bottom of the screen. Rendered outside the scroll container via sibling placement (not child). Four tabs: Today, History, Nurse, Settings.

| State | Visual |
|---|---|
| Default | Icon + label in `--t2`, `fontWeight: 400` |
| Active | Icon + label in `--acc`, `fontWeight: 600` |
| Nurse (new tips) | Shows a dot badge in `--acc` when `hasNewTips` is true |

**Constraints:** `position: fixed`. Background `--card`. Border top `0.5px solid --sep`. Shadow `-1px 12px rgba(0,0,0,0.07)` upward. Padding bottom `env(safe-area-inset-bottom)`.

---

### SessionGroup (GroupCard)

Container for a scheduled medication session (Morning, Lunch, Evening). Wraps pill cards.

| State | Visual |
|---|---|
| Has pills | Section label + progress indicator + pill list |
| All taken | Progress label shows "✓ Done" in `--green` |
| Partial | Progress shows "N / total" count |
| Empty (no pills in session) | Section is hidden entirely — not rendered |

**Structure:** Label row (uppercase, `--t1`, `fontWeight: 600`, `letterSpacing: 0.04em`) + progress count right-aligned. Card list below with `gap: 10px`.

---

### PillCard

Individual medication row inside a SessionGroup. Tappable — opens PillDetailSheet.

| State | Visual |
|---|---|
| Untaken | Full opacity (1.0). Checkmark circle: empty border in `--sep`. |
| Taken | Opacity 0.55. Checkmark circle: filled with session accent color + white IconCheck. |
| Flash (just tapped) | `.pop` animation (scale bounce) on the checkmark button. |
| Expiring soon (≤3 days) | Duration label shows "N days left" in `--acc`. |
| Expiring later | Duration label shows "Until [date]" in `--t3`. |

**Structure:** Left icon area (68×92px, `--acc-soft` background, ShapeIcon in `--acc`) + center text block (name in `--t1` 16px semibold, dose in `--t2` 14px, timing/duration in `--t2` 13px) + right checkmark button (44×44px tap target, 28×28px circle).

**Constraints:** `borderRadius: 18px`. `--shadow-md`. `overflow: hidden`. Entry animation: `pillIn` keyframe (fade up + scale from 0.98), staggered by 60ms per card.

---

### AsNeededCard

Medication row for as-needed drugs. Different interaction model — tapping the + button logs a timestamped dose instead of toggling a boolean.

| State | Visual |
|---|---|
| Available (canTake) | + button in `--acc`. No status badge. |
| Taken today (count > 0) | Green label: "Nx today · last at HH:MM". |
| Interval blocked | Blue chip: "Next at HH:MM". |
| Max reached | Orange warning chip: "Max Nx reached" with triangle icon. |

**Constraints:** No opacity dimming (unlike PillCard). The card is always full opacity. The + button is the only action — tap on the card itself opens the detail sheet.

---

### InventoryBadge

Inline badge on PillCard or DetailSheet showing remaining pill count.

| State | Visual |
|---|---|
| Normal stock | Not shown |
| Low stock (≤5 remaining, renewable only) | Gold badge: remaining count + pill icon |
| Renewable | Violet badge: "Renewable" |
| Non-renewable, out | No badge (not tracked) |

**Constraints:** Badge uses `--gold` / `--gold-bg` for low stock. `--violet` / `--violet-bg` for renewable. Font size 12px, `fontWeight: 700`, `borderRadius: 6px`, padding `2px 8px`.

---

### PillForm (BottomSheet)

Full-screen form sheet for adding or editing a medication. Opens via BottomSheet.

| State | Visual |
|---|---|
| Add mode | Submit button label "Add", accent color from session |
| Edit mode | Submit button label "Save" |
| Saving | Button disabled, loading state (not yet implemented — use opacity 0.6 if added) |
| Drug name — idle | Plain text input, no suggestions |
| Drug name — typing (≥2 chars) | OpenFDA autocomplete dropdown appears below input |
| Drug name — loading suggestions | No visual (debounced 300ms, silent fetch) |
| Drug name — has results | Dropdown list with drug names, tappable rows |
| Drug name — no results | Dropdown hidden, input retains typed value |
| Scan label | Camera icon button. Loading state: `scanLoading` disables button. |
| Duration — ongoing | Toggle on, stepper hidden |
| Duration — fixed | Toggle off, stepper visible (days) |
| Inventory — enabled | Start date + count + renewable toggle visible |
| Inventory — disabled | Section hidden |

**Constraints:** `noAutoFocus` prop prevents keyboard on open. `requestAnimationFrame` resets scroll to top on mount. Sheet enters with `sheetUp` keyframe (translateY from 100%). Form uses `type="text" inputMode="decimal"` on numeric inputs (not `type="number"` — iOS cursor bug).

---

### BottomSheet

Generic draggable overlay. Used by PillForm, PillDetailSheet, and any modal content.

| State | Visual |
|---|---|
| Open | Sheet enters from bottom with `sheetUp` animation. Background overlay. |
| Dragging | Sheet follows touch Y position (only when scrollTop = 0). |
| Dismissing | Snap back or dismiss threshold. `onClose` called on dismiss. |

**Constraints:** `ReactDOM.createPortal` into `document.body` — required to escape `overflow: hidden` on parent cards. Touch handling: horizontal scroll inside sheet (`data-scroll-x`) locks out drag dismiss to avoid conflict.

---

### HistoryView

30-day log of medication adherence. Secondary screen, accessed via tab.

| State | Visual |
|---|---|
| List view (default) | Scrollable list of days, each showing a completion ring + date + score |
| Day selected | Detail view for that day — all active pills with taken/missed status |
| Day fully complete | Ring filled in `--green`, score "✓ Done" |
| Day partial | Ring partially filled in `--acc`, score "N / total" |
| Day empty (no log) | Not shown in the list |
| Loading more | "Load all history" button in `--blue`, shows spinner while fetching |
| Archive event on a date | Pill event card (expired / stopped / changed) shown inline in the day |

**Constraints:** Default shows last 30 days. "Load all" fetches full Supabase history. Past-day log toggling is available in day detail view via the same checkmark UI as Today.

---

### HealthAssistant (NurseTab)

Chat interface with AI nurse persona. Persistent across sessions via `nurseMsgs` state in App.

| State | Visual |
|---|---|
| Empty (no messages) | Suggestion chips shown (3 preset questions) |
| Loading response | Animated dots (3 bouncing dots with `bounce` keyframe) |
| Response received | Assistant bubble in `--bg2`, user bubble in `--acc-soft` |
| Error | Red error message with retry button |
| Keyboard open | `visualViewport` listener adjusts bottom padding to keep input visible |

**Constraints:** Known design debt — robot avatar and bouncing dots create a generic chatbot feel that conflicts with the "trusted nurse" persona. Suggestion chips and avatar are queued for a warmth redesign. Disclaimer shown once only (not three times — see Known Design Debt). Input bar is portal-rendered to avoid iOS `position: fixed` keyboard trap.

---

### DrugAutocomplete

Inline suggestion list inside PillForm, below the drug name input.

| State | Visual |
|---|---|
| Idle | Hidden |
| Typing (≥2 chars) | Dropdown appears, fetches from OpenFDA |
| Loading | Silent (debounced 300ms, no spinner) |
| Has results | List of tappable drug names |
| No results | Dropdown hidden |
| Suggestion selected | Input filled, dropdown dismissed |

**Constraints:** Rendered via `ReactDOM.createPortal` to escape `overflow: hidden` on the form card. Debounce: 300ms. Min query length: 2 chars.

---

### ShapeIcon

Inline SVG pill shape renderer. Used on PillCard, AsNeededCard, and PillForm shape carousel.

| State | Visual |
|---|---|
| Known shape | Renders matching SVG (Capsule, Round, Oval, Oblong, Mini Round, Diamond) |
| Unknown shape | Falls back to Capsule |
| Shape carousel (PillForm) | Horizontally scrollable row of 6 shape options, selected shape highlighted in `--acc` |

**Constraints:** 6 shapes supported. `data-scroll-x` attribute on carousel to prevent drag-dismiss conflict with BottomSheet. Color prop accepted — use `var(--acc)` for active state, `var(--t2)` for inactive.

---

## Known design debt

Captured from an impeccable critique. Not yet fixed.

1. **`--t3` on small text fails contrast.** Tertiary text on tinted near-white lands around 1.9:1, well under the 4.5:1 needed for body. Rule: never put readable text in `--t3`. Use `--t2` as the floor. Audit any small label, disclaimer, or caption currently in `--t3`.
2. **The warm-paper background reads as "AI default" in 2026.** It is committed brand identity, so it stays. The discipline: do not add new cream or ivory tinted surfaces. Carry warmth through `--acc` and type, not more beige.
3. **Nurse tab uses the most generic chatbot archetype** (robot avatar, three bouncing dots, suggestion chips). Tonal mismatch with the "caring nurse" promise. Queued to warm up.
4. **Nurse tab repeats its disclaimer three times** (subtitle, banner, per-message footer). Keep one strong placement only.
5. **Nurse suggestion "Can I stop early?"** steers users toward a question the assistant is forbidden to answer. Replace with safer prompts.
