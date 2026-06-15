# Posology — Technical Specs

Pure technical reference. How the app is built. For behavioral rules and commit gates, see **CLAUDE.md**. For product intent, see **PRD.md**.

## Stack

- React 18 (CDN/UMD) + Babel standalone for in-browser JSX
- **Supabase** for cloud persistence — `sbGet`/`sbSet` only, never IndexedDB
- CSS custom properties, auto dark/light via `prefers-color-scheme`
- Service worker for offline support and home screen PWA caching
- Single `index.html` — everything lives here
- Supabase Edge Functions: `get-tips` (AI recommendations), `nurse-chat` (Nurse tab)

---

## Storage

- **Supabase project:** `https://fotzkqkghxndjnomrspr.supabase.co`
- **Anon key:** embedded in `index.html` (public by design)
- **Access:** `?u=username` URL param. The token is the namespace, no auth or registration required
- **KV schema:** `(user_token, key, value jsonb)`
- **Upserts:** must use `Prefer: resolution=merge-duplicates`
- **SW fetch handler:** must bypass cache for all `supabase.co` requests

**Token setup for new users:** share `https://cff.github.io/pill-reminder/?u=theirname`. The Supabase namespace is created automatically on first write. Nothing to configure in Supabase.

To verify in the Supabase dashboard (cannot be tested from Claude's environment): UNIQUE constraint on `(user_token, key)`, and Row Level Security policies filter by token.

---

## Data model

| Supabase key | Value | Notes |
|---|---|---|
| `pr:list` | `{ morning: [...], lunch: [...], evening: [...], as_needed: [...] }` | Pill definitions |
| `pr:day:YYYY-MM-DD` | `{ morning: {id: bool}, lunch: {id: bool}, evening: {id: bool} }` | Scheduled dose logs |
| `pr:asneeded:YYYY-MM-DD` | `{ [pill_id]: [timestamp_ms, ...] }` | As-needed dose timestamps per day |
| `pr:archived` | `{ morning: [...], lunch: [...], evening: [...] }` | Pills with endDate |
| `pr:profile` | `{ name, age, weight, height }` | User profile |
| `pr:tips:HASH` | Cached AI tips array | Keyed by hash of pill list description |

**localStorage (device-only, not Supabase):**
- `pr:seenTipLabels` — tip labels scrolled past (suppresses "New" badge on reload)

---

## Architecture

- Single `App` component holds all state
- All named sub-components defined **before** `App` — required by Babel standalone hooks enforcement
- `TabBar` renders outside the scrollable container (sibling, not child) — required for `position:fixed`
- Dropdowns use `ReactDOM.createPortal` into `document.body` — required to escape `overflow:hidden` on `GroupCard`
- Today scroll container has class `today-scroll`
- Nurse chat header: `position:sticky; top:0; zIndex:50`
- Nurse chat state (`nurseMsgs`) is held in `App`, passed into `HealthAssistant`, so the conversation survives tab switches

---

## Pill model

```
{ id, name, dose, shape, schedules: [{session, timing}],
  startDate, durationDays, asNeeded, maxDailyDose, minIntervalHours,
  dosageAmt, dosageUnit, hasInventory, inventoryCount, renewable }
```

---

## Design tokens

> Planned to migrate to DESIGN.md when that file is created.

`var(--acc)` orange — in-progress
`var(--acc-soft)` — light orange background
`var(--green)` — fully complete
`var(--blue)` — new / informational
`var(--red)` — destructive
`var(--t1)` / `var(--t2)` — primary / secondary text (`--t3` too low contrast, avoid on small text)
`var(--sep)` / `var(--sep-strong)` — separators (use strong for disclaimers and tip card borders)
`var(--card)` / `var(--bg)` / `var(--bg2)` — surfaces

Fonts: DM Serif Display (titles) + system font (body)
Pill cards: border-radius 18px. Group cards: 16px.
iOS HIG: 44px tap targets, `env(safe-area-inset-*)` for all edges.
