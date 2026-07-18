# Feature PRD: As-needed medications

> Derived from: [PRD.md](PRD.md)
> Any principle not explicitly overridden here follows the product PRD.
> Reverse-engineered from `index.html` at pr-v194 (dev).

## Overview

Tracks timestamped doses for PRN (pro re nata / "as needed") medications such as pain relief, where the user decides in the moment whether to take a dose rather than following a fixed schedule. The feature exists to prevent dangerous double-dosing while staying out of the way for a healthy day.

## Problem this feature solves

Scheduled sessions (Morning/Lunch/Evening) assume a fixed routine. As-needed drugs don't fit that model — the risk isn't "did I forget," it's "did I take too much too soon." This feature enforces safety limits (daily max, minimum interval) without requiring the user to do mental math at 2am.

## Users

Same as product PRD. No feature-specific segment.

## Acceptance criteria

- [ ] Taking a dose increments the today count and updates "last at HH:MM" immediately
- [ ] A second dose within `minIntervalHours` is blocked, button shows "Next at HH:MM"
- [ ] Reaching `maxDailyDose` disables the button and shows "Max Nx reached"
- [ ] A dose taken at 23:54 correctly blocks a new dose after midnight until the interval elapses, but does not appear in today's displayed count or dose list
- [ ] "Add missed dose" for yesterday updates the interval guard without writing yesterday's timestamp into today's Supabase record
- [ ] Editing or deleting a timestamp in the detail sheet re-derives today's data cleanly, no orphaned entries
- [ ] Deleting the pill entirely clears its entry from pill list, today's log, and today's timestamp map, in Supabase

## Functional requirements

**Core state:** `asNeededTs[pill_id]` is a sorted array of millisecond timestamps, held in React state and mirrored to two Supabase keys: `pr:asneeded:YYYY-MM-DD` (all doses for that calendar day) and folded into `pr:day:YYYY-MM-DD` under `as_needed`.

**The 24h window.** On load, `asNeededTs` is merged from today's Supabase record plus any late timestamps from yesterday still relevant to the interval guard (see Data reads/writes). This in-memory array can therefore span two calendar days. `todayOnlyTs()` filters it down to the current calendar day wherever a UI element must show only today's activity.

**Behavioral contracts — source of truth per element:**

| Element | Source | Scope | Code reference |
|---|---|---|---|
| "Nx today" count | `todayOnlyTs(timestamps).length` | Calendar day | `asNeededStatus()` |
| "last at HH:MM" | `todayTs[count-1]` where `todayTs = todayOnlyTs(timestamps)` | Calendar day | Today card render |
| "Next at HH:MM" / button blocked | `timestamps[timestamps.length-1]` (full 24h array) | 24h window | `asNeededStatus()` |
| "Max Nx reached" | `count >= pill.maxDailyDose`, count is today-only | Calendar day | `asNeededStatus()` |
| Detail sheet dose list | `todayOnlyTs(timestamps)` | Calendar day | Detail sheet render |
| Interval check (`canTake`) | `!atMax && intervalOk`, where `intervalOk = !nextAllowedTs \|\| Date.now() >= nextAllowedTs` | 24h window | `asNeededStatus()` |

**Exact logic** (from `asNeededStatus(timestamps, pill)`):
```
todayArr = todayOnlyTs(timestamps)
count = todayArr.length
atMax = pill.maxDailyDose && count >= pill.maxDailyDose
if timestamps is empty: canTake = true, nextAllowedTs = null
lastTs = timestamps[timestamps.length - 1]        // full 24h array, not today-only
intervalMs = (pill.minIntervalHours || 0) * 3600000
nextAllowedTs = intervalMs > 0 ? lastTs + intervalMs : null
intervalOk = !nextAllowedTs || now >= nextAllowedTs
canTake = !atMax && intervalOk
```

**Write actions:**

| Action | What happens |
|---|---|
| `takeAsNeeded(id)` | Appends `Date.now()` to `asNeededTs[id]`, dedupes via `Set`, sorts. Writes full map to `pr:asneeded:today` (today-only, via `cleanMapToday`) and folds into `pr:day:today`. Triggers a brief flash animation on the card. |
| `updateTimestamps(newTs)` (detail sheet edit/delete) | Replaces the array for one pill, dedupe + sort, writes today-only to both keys. Same shape as `takeAsNeeded`, no special-casing for edit vs delete. |
| `addMissedDose(dtStr)` | If date = today: same path as `takeAsNeeded`. If date = yesterday: reads `pr:asneeded:yesterday`, merges the new timestamp into that day's own record (not today's), then separately updates today's in-memory `asNeededTs` state so the interval guard sees it — but does **not** write yesterday's timestamp into today's Supabase record. |
| `deleteAsNeededPill(id)` | Removes the pill from `pr:list`, deletes its entry from today's log and today's timestamp map. Triggers tip regeneration. |

## Data reads and writes

**Supabase keys:**
- `pr:asneeded:YYYY-MM-DD` — `{ [pill_id]: [timestamp_ms, ...] }`, one record per calendar day, always today-only content
- `pr:day:YYYY-MM-DD` — as-needed doses also folded into `log.as_needed[pill_id]`, same today-only content, kept in sync with the above on every write
- `pr:list` — pill definitions (`maxDailyDose`, `minIntervalHours`, etc.), read on load

**On load:** the app reads today's `pr:asneeded:` record, then separately reads yesterday's record and filters it to timestamps recent enough to still matter for the interval guard (i.e. within `minIntervalHours` of now, conceptually — implemented as the general 24h merge). This merged result becomes the in-memory `asNeededTs[pill_id]` array. The yesterday-derived entries are never persisted into today's Supabase record and never rendered in today-scoped UI.

**localStorage:** none specific to this feature (writes are Supabase-only, per project storage rule).

## Edge cases and empty states

- **No `minIntervalHours` set:** `intervalMs = 0`, so `nextAllowedTs = null`, interval check always passes. Only the daily max applies.
- **No `maxDailyDose` set:** `atMax` is always false. Only the interval applies.
- **Neither set:** button is always available, feature degrades to a plain "log a dose" timestamp list.
- **Midnight rollover while app is open:** not actively re-evaluated on a timer; `todayOnlyTs()` is a pure function of `Date.now()` called on render, so a stale-open tab will naturally recompute correctly on next interaction/render, but there is no polling to force a re-render at exactly midnight.
- **Dose taken, then app closed and reopened next day:** yesterday's dose is preserved for the interval guard only, via the merge-on-load step. Correct.
- **Editing a timestamp to a future time:** not blocked client-side as far as inspected code shows. Not a documented constraint.

## Must match exactly vs UX can change

**Must match exactly** (rebuild in any UI must preserve this):
- The today-only vs 24h-window split described in the behavioral contracts table. This is the single most bug-prone part of the app (see PRD.md Change protocol — a past bug indexed into the wrong array).
- `addMissedDose` never writing a cross-day timestamp into the wrong day's Supabase record.
- Dedup + sort on every write to the timestamp array.
- The Supabase key shapes (`pr:asneeded:YYYY-MM-DD`, `as_needed` sub-object in `pr:day:YYYY-MM-DD`) — required for a rebuilt UI to read the same user data with zero migration.

**UX can change:**
- Card layout, flash animation on take, button copy ("Next at HH:MM" phrasing)
- Detail sheet presentation (bottom sheet vs modal vs inline)
- Missed-dose picker UI (currently datetime-local input, min=yesterday 00:00, max=now)

## Out of scope

- Any UI or logic for scheduled (Morning/Lunch/Evening) medications — covered by a separate feature
- Push notifications when the interval elapses (not implemented anywhere)
- Retroactive editing of doses from more than 1 day ago (missed-dose picker is capped at yesterday)

## Open questions

- Should the interval guard poll a timer to auto-unblock the button in real time, or is "correct on next render" an accepted tradeoff? Not raised as a bug so far — leave as is unless it becomes one.
