# Posology — Project Context for Claude

This file is how Claude operates on this project: the gates, the rules, and the guardrails to check before editing. Pure technical reference lives in **SPECS.md**. Product intent lives in **PRD.md**.

## What this is

A personal daily medication tracker built as a single-file PWA (`index.html`), hosted on GitHub Pages. Installed via "Add to Home Screen" on iPhone. No build step, no CLI, no tooling, just a file.

**Live URL:** <https://cff.github.io/pill-reminder/>
**Dev URL:** <https://cff.github.io/pill-reminder-dev/>

Stack in one line: React 18 (CDN/UMD) + Babel standalone, Supabase for persistence (`sbGet`/`sbSet` only, never IndexedDB), service worker for PWA caching, everything in one `index.html`. Full stack and schema in SPECS.md.

---

## COMMIT GATE — run before every commit

Before any PUT to GitHub, answer each out loud. If any answer is "no", stop.

1. **Dev first?** Is this change going to `dev` first? A prod commit only happens after the change is on dev and a dev↔prod diff has been run.
2. **Approved?** Did Claire give explicit approval for *this* commit?
3. **Fresh?** Was the file and its SHA fetched in *this* step, not reused from earlier in the session?
4. **Valid?** Was the JSX validated with `@babel/parser` (for `index.html` changes)?
5. **Bumped?** Is the version string `pr-vNN` bumped, with count == 2?

"It's small and already validated" is not an exception. That is exactly the situation where the dev-first rule was skipped once. The gate exists to catch momentum, not ignorance.

---

## CRITICAL WORKFLOW RULES

### 0. Orient before proposing

At the start of a work session, before proposing any change: note the current version (`pr-vNN`) of **both** dev and prod, and flag any feature divergence between them. Do not operate from a remembered mental map of repo state, it goes stale. One past session began with a wrong assumption about which features were where.

### 1. Always fetch fresh before patching

Never reuse a previously fetched file from earlier in the session.

1. Fetch file + SHA via GitHub Contents API (JSON response, base64 content)
2. Verify version with `grep -o "pr-v[0-9]*"`
3. Apply patches in Python on the in-memory string
4. Validate JSX with `@babel/parser` before committing
5. Fetch fresh SHA immediately before PUT, never reuse a cached SHA
6. Commit via PUT with fresh SHA

Two past incidents (data loss at v57, blank page at v81) both came from patching a stale copy.

### 2. Dev before prod — always

- All changes go to `cff/pill-reminder-dev` first
- Test on dev, then port **surgically** to prod, never overwrite prod with a full dev copy
- Before claiming a port is complete, run a **systematic line-by-line diff** between dev and prod
- Dev and prod have independent version strings (dev is always ahead)

### 3. Preview before commit

Show a diff or UI preview and wait for explicit approval before committing. Exception: Claude may skip visual preview for pure bug fixes if the change is < 5 lines and the logic is unambiguous, but must still say what is changing and why.

### 4. JSX validation is mandatory

Extract the `<script type="text/babel">` block, parse with `@babel/parser` (plugins: jsx), abort if it fails. A missing closing `</div>` has caused a blank page before.

### 5. Large file — use Python, not bash

File is ~124KB. Bash heredocs fail silently at this size. Always use Python for reading, patching, and base64-encoding before commit.

### Versioning

Bump the SW cache string and the Settings version string on every meaningful commit. Exactly 2 occurrences, use global replace, verify count == 2. Dev and prod version numbers are independent and dev is always ahead, so do not pin specific numbers in docs (they go stale).

---

## As-needed dose tracking — data invariants

This is the most complex part of the app and the source of past bugs. Read carefully before touching anything related to `asNeededTs`, `takeAsNeeded`, `asNeededStatus`, or the detail sheet. These are behavioral guardrails, not reference, which is why they live here and not in SPECS.md.

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

> If you change how `count` is derived, audit every place `timestamps[count-1]` or `timestamps[X]` is used as an index. The index may be into a different-sized array.

---

## Debugging discipline

When a fix "still doesn't work," resist patching again. Diagnose first.

- **Reason from what's on screen, not from confidence.** Find something that *works*, compare it to what's broken, and the difference localizes the cause. The scroll-button bug was solved by noticing the tab bar stayed fixed while the input did not. The only difference was being inside the animated container.
- **Find the one root cause before touching symptoms.** Fixing a threshold, then a transform, then an animation one at a time treats branches, not the root. Each patch can look right and still miss it.
- **Verify the new code is actually running before concluding a fix failed.** A stale service-worker cache served old versions for several rounds and made correct fixes look broken. Now mitigated: the SW is network-first for navigations.
- **Hold confidence loosely.** Present a cause as a hypothesis and let the user confirm on-device before declaring it fixed. Claude cannot see the screen; the user can.

---

## Known gotchas

- **Babel hooks rules:** no hooks after conditional returns, no components defined inside `App`
- **`overflow:hidden` clips dropdowns:** use `ReactDOM.createPortal`
- **GitHub Secret Scanning** blocks `sk-ant-api03-*` even in comments. Anthropic key lives in Supabase secrets only
- **SW cache is isolated from Safari.** Version bump is mandatory on every meaningful commit
- **Version string appears in exactly 2 places.** Verify count == 2 after every replace
- **Dev-only icons:** The splash screen double-pill SVG and the PWA manifest favicon are intentionally different from prod. Do not port them to `cff/pill-reminder`. They exist solely to make dev visually identifiable at a glance.
- **CSS transforms break `position:fixed` for descendants**, including a transform left behind by an animation's fill mode. Elements that must stay put (Assistant input, scroll button) are portaled to `body` to escape the `fade-up` container

---

## Doc map

- **CLAUDE.md** (this file): how Claude operates. Gates, workflow rules, data invariants, gotchas.
- **PRD.md**: what the product is and why. Vision, users, features, scope, product change protocol.
- **SPECS.md**: technical reference. Stack, storage schema, architecture, data model, pill model.
- **DESIGN.md**: visual system. Identity, color, typography, component conventions, known design debt.
