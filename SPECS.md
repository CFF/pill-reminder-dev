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
- **Access:** `?u=name-XXXXXXXXXX` URL param (high-entropy token). The token is the namespace; no auth or registration
- **KV schema:** `(user_token, key, value jsonb)`
- **Upserts:** must use `Prefer: resolution=merge-duplicates`
- **SW fetch handler:** must bypass cache for all `supabase.co` requests

**Token setup for new users:** generate a high-entropy token (`name-XXXXXXXXXX`, 10 random chars) and share `https://cff.github.io/pill-reminder/?u=<that token>`. The namespace is created on first write. Do not issue guessable first-name tokens.

**Access path (locked, 2026-07):** the `data` table is not directly reachable with the anon key. All reads and writes go through `SECURITY DEFINER` functions, each scoped to the token passed in: `get_one(p_token, p_key)`, `get_prefix(p_token, p_prefix)`, `set_one(p_token, p_key, p_value)`. Direct anon `SELECT/INSERT/UPDATE` on `public.data` is revoked and RLS is enabled, so `/rest/v1/data` returns permission-denied to the public key. Never query the table directly — use or extend these functions.

Dashboard checks (cannot be tested from Claude's environment): UNIQUE constraint on `(user_token, key)` still required for upserts; a dump attempt with the anon key (`GET /rest/v1/data`) must return permission-denied.

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
- Root render is wrapped in `ErrorBoundary` (class component, defined before `App`) — render crashes show a reload fallback instead of a blank page

---

## Edge functions

Two Supabase Edge Functions proxy the Anthropic API, which keeps the key server-side and bypasses CORS:
- `nurse-chat`: powers the Assistant chat
- `get-tips`: generates the AI dietary tips, cached by pill-list hash

Both call model `claude-sonnet-4-6`, hardcoded in each function. When a model is retired, the string must be updated on Supabase. Sonnet 4.0 (`claude-sonnet-4-20250514`) retired 2026-06-15 and broke both functions until updated. The source is backed up in the prod repo (`cff/pill-reminder`) under `supabase/functions/` — the dev repo does not carry a copy — but the backup does not auto-deploy. Live edits happen on the Supabase dashboard.

**Hardening in place (audit 2026-07):** both functions guard for a missing `ANTHROPIC_API_KEY`, wrap parsing and the upstream fetch in error handling, and restrict CORS to `https://cff.github.io`. `nurse-chat` additionally clamps `max_tokens` server-side, prepends a locked system prefix before any client-supplied `system`, and caps request body size. Any future edit to these functions must preserve all of the above.

---

## Pill model

```
{ id, name, dose, shape, schedules: [{session, timing}],
  startDate, durationDays, asNeeded, maxDailyDose, minIntervalHours,
  dosageAmt, dosageUnit, hasInventory, inventoryCount, renewable }
```

---

## Design system

Color tokens, typography, component conventions, and known design debt now live in **DESIGN.md**.
