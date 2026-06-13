# Posology — Project Context for Claude

## What this is

A personal daily medication tracker built as a single-file PWA (`index.html`), hosted on GitHub Pages. Installed via "Add to Home Screen" on iPhone. No build step, no CLI, no tooling — just a file.

**Live URL:** <https://cff.github.io/pill-reminder/>  
**Dev URL:** <https://cff.github.io/pill-reminder-dev/>

## Stack

- React 18 (CDN/UMD) + Babel standalone for in-browser JSX
- **Supabase** for cloud persistence — `sbGet`/`sbSet` only, never IndexedDB
- CSS custom properties, auto dark/light via `prefers-color-scheme`
- Service worker for offline support and home screen PWA caching
- Single `index.html` — everything lives here
- Supabase Edge Functions: `get-tips` (AI recommendations), `nurse-chat` (Nurse tab)

## Current version

`pr-v105` (prod) / `pr-v107` (dev). Bump SW cache string and Settings version string on every commit — exactly 2 occurrences, use global replace, verify count == 2.

---

## CRITICAL WORKFLOW RULES

### 1. Always fetch fresh before patching

Never reuse a previously fetched file from earlier in the session.

1. Fetch file + SHA via GitHub Contents API (JSON response, base64 content)
2. Verify version with `grep -o "pr-v[0-9]*"`
3. Apply patches in Python on the in-memory string
4. Validate JSX with `@babel/parser` before committing
5. Fetch fresh SHA immediately before PUT — never reuse a cached SHA
6. Commit via PUT with fresh SHA

Two past incidents (data loss at v57, blank page at v81) both came from patching a stale copy.

### 2. Dev before prod — always

- All changes go to `cff/pill-reminder-dev` first
- Test on dev, then port **surgically** to prod — never overwrite prod with a full dev copy
- Before claiming a port is complete, run a **systematic line-by-line diff** between dev and prod
- Dev and prod have independent version strings (dev is always ahead)

### 3. Preview before commit

Show a diff or UI preview and wait for explicit approval before committing. Exception: Claude may skip visual preview for pure bug fixes if the change is < 5 lines and the logic is unambiguous — but must still say what is changing and why.

### 4. JSX validation is mandatory

Extract the `<script type="text/babel">` block, parse with `@babel/parser` (plugins: jsx), abort if it fails. A missing closing `</div>` has caused a blank page before.

### 5. Large file — use Python, not bash

File is ~124KB. Bash heredocs fail silently at this size. Always use Python for reading, patching, and base64-encoding before commit.

---

## Storage

- **Supabase project:** `https://fotzkqkghxndjnomrspr.supabase.co`
- **Anon key:** embedded in `index.html`
- **Access:** `?u=username` URL param — token is the namespace, no auth/registration required
- **KV schema:** `(user_token, key, value jsonb)`
- **Upserts:** must use `Prefer: resolution=merge-duplicates`
- **SW fetch handler:** must bypass cache for all `supabase.co` requests

**Token setup for new users:** share `https://cff.github.io/pill-reminder/?u=theirname` — Supabase namespace is created automatically on first write. Nothing to configure in Supabase.

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

## As-needed dose tracking — data invariants

This is the most complex part of the app and the source of past bugs. Read carefully before touching anything related to `asNeededTs`, `takeAsNeeded`, `asNeededStatus`, or the detail sheet.

### State variables

| Variable | Contents | Purpose |
|---|---|---|
| `asNeededTs[pill_id]` | Sorted timestamp array, **24h window** (today + yesterday recent) | Interval guard across midnight |
| `todayOnlyTs(asNeededTs[pill_id])` | Filtered to current calendar day | Display and counting |

### Why the 24h window exists

If a dose was taken at 23:54 yesterday and the minimum interval is 6h, the app must know about it at 1:00 AM to block the button. The merge on load pulls recent yesterday timestamps for this purpose only. They are **never displayed** to the user and **never persisted** in today's Supabase record.

### Behavioral contracts — what each element reads

| UI element | Data source | Rule |
|---|---|---|
| Today card "Nx today" count | `todayOnlyTs(timestamps).length` | Today only |
| Today card "last at HH:MM" | `todayTs[count-1]` where `todayTs = todayOnlyTs(timestamps)` | Today only |
| Today card "Next at HH:MM" / button blocked | `timestamps[timestamps.length-1]` (full 24h) | Full window |
| Today card "Max Nx reached" | `count >= pill.maxDailyDose` where count is today-only | Today only |
| Detail sheet dose list | `displayTs = todayOnlyTs(timestamps)` | Today only |
| `asNeededStatus()` count | `todayOnlyTs(timestamps).length` | Today only |
| `asNeededStatus()` interval check | `timestamps[timestamps.length-1]` (full 24h) | Full window |

### Write rules

- `takeAsNeeded` — deduplicate, persist today-only via `cleanMapToday()` to both `pr:asneeded:today` and `pr:day:today`
- `updateTimestamps` (detail sheet edit/delete) — same: deduplicate, persist today-only
- `addMissedDose` — if date is today: normal path. If date is yesterday: write to `pr:asneeded:yesterday`, update today's `asNeededTs` state (for interval guard), do NOT write yesterday's ts into today's Supabase record

### Invariant to check before any patch touching this system

> If you change how `count` is derived, audit every place `timestamps[count-1]` or `timestamps[X]` is used as an index — the index may be into a different-sized array.

---

## Architecture

- Single `App` component holds all state
- All named sub-components defined **before** `App` — required by Babel standalone hooks enforcement
- `TabBar` renders outside the scrollable container (sibling, not child) — required for `position:fixed`
- Dropdowns use `ReactDOM.createPortal` into `document.body` — required to escape `overflow:hidden` on `GroupCard`
- Today scroll container has class `today-scroll`
- Nurse chat header: `position:sticky; top:0; zIndex:50`

## Pill model

```
{ id, name, dose, shape, schedules: [{session, timing}],
  startDate, durationDays, asNeeded, maxDailyDose, minIntervalHours,
  dosageAmt, dosageUnit, hasInventory, inventoryCount, renewable }
```

## Design tokens

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

## Known gotchas

- **Babel hooks rules:** no hooks after conditional returns, no components defined inside `App`
- **`overflow:hidden` clips dropdowns:** use `ReactDOM.createPortal`
- **GitHub Secret Scanning** blocks `sk-ant-api03-*` even in comments — Anthropic key in Supabase secrets only
- **SW cache is isolated from Safari.** Version bump is mandatory on every meaningful commit
- **Version string appears in exactly 2 places.** Verify count == 2 after every replace
