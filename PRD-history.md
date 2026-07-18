# Feature PRD: History

> Derived from: [PRD.md](PRD.md)
> Any principle not explicitly overridden here follows the product PRD.
> Reverse-engineered from `index.html` at pr-v194 (dev).
> Supersedes PRD.md section 3 — UC3 there is marked "not yet implemented" but is implemented (see Retroactive correction below). This file is the source of truth going forward.

## Overview

A scrollable log of past days, each showing which scheduled pills were taken or missed, an as-needed dose list, a compliance score, and now a running timeline of medication changes (started/stopped/dose changed). Lets Claire correct a mis-tap after the fact and, when needed, show a doctor an accurate record.

## Problem this feature solves

Today's view is transient by design, you check pills off and move on. History is the accountability layer: it has to reconstruct what was actually prescribed on any given past day, even if the pill has since been edited, versioned, or archived, and it has to let a genuine mistake (mis-tap, forgot to log) be corrected without corrupting the record for other days.

## Users

Same as product PRD. No feature-specific segment.

## Acceptance criteria

- [ ] Opening History shows the last 30 days, most recent first
- [ ] Each day shows only pills that were actually active on that date, using archived pill versions where relevant, not today's current pill list
- [ ] Sessions with zero active pills for that day are hidden entirely
- [ ] Tapping a pill's taken/missed state in History toggles it and persists immediately
- [ ] The day's compliance score (percentage) updates immediately after a toggle
- [ ] As-needed doses for that day show their logged timestamps, read-only in the current build
- [ ] The events timeline shows medication start/stop/dose-change entries in reverse-chronological order
- [ ] History remains correct after a pill has been versioned (edited clinically) — the old dose shows for old dates, the new dose for new dates

## Functional requirements

### Active-pill reconstruction (`buildActivePillsForDate`)

For a given date, walks current `pr:list` and `pr:archived` and returns every pill session-pair that was active on that date:
- **Current pills** (still in `pr:list`): included if `date >= startDate`, and if `durationDays` is set, only while `date < startDate + durationDays`. No `durationDays` means ongoing, included for any date `>= startDate`.
- **Archived pills**: included if `startDate <= date < endDate`. This is what makes a versioned pill's old dose show correctly on old dates, since the archived copy carries the pre-edit values.

### Compliance score (`dayScore`)

```
pills = buildActivePillsForDate(date, list, archived)   // or today's live list if date is null
total = pills.length
taken = count of pills where log[pill.session][pill.id] is true
pct = total ? round(taken / total * 100) : 0
```
Purely derived, not stored. Recomputed on every render from the day's log plus the active-pill set.

### Retroactive correction (`toggleHistoryTaken`)

Fetches the log for the given past date, flips the boolean for that session+pill, writes it back to `pr:day:YYYY-MM-DD`, and updates the in-memory `history` array for that date. No confirmation step, no undo, immediate write. This is a full read-modify-write, not a merge, so it correctly handles a day whose log document didn't exist yet.

### As-needed doses in History

Sourced from `pr:asneeded:YYYY-MM-DD` for that day, displayed as a read-only timestamp list per pill. No edit affordance in History itself (editing as-needed timestamps happens via the Today detail sheet, see PRD-as-needed.md — that sheet is scoped to today only).

### Events timeline

A flat array, `pr:events`, of entries prepended (newest first) on three triggers:

| Event type | Fields | Triggered by |
|---|---|---|
| `added` | `date, pillName, dose, session, pillId` | New pill saved (manual add or scan) |
| `stopped` | `date, pillName, dose, session, pillId` | Pill deleted |
| `changed` | `date, pillName, oldDose, newDose, session, pillId` | Clinical edit confirmed (version change) |

Not scoped per day, it's one array for the whole account, rendered as a timeline alongside or within History. Included in JSON backup export/import (`data.events`).

## Data reads and writes

**Supabase keys:**
- `pr:day:YYYY-MM-DD` — read for each of the last 30 days on History load; read-modify-write on toggle
- `pr:asneeded:YYYY-MM-DD` — read for each of the last 30 days, display only
- `pr:archived` — read once, used for active-pill reconstruction on any past date
- `pr:list` — read once, used for active-pill reconstruction on current dates
- `pr:events` — read once on load into `storedEvents` state; every write elsewhere in the app (add/edit/delete pill) also writes here, so History always reflects the latest without a manual refresh within a session

**Write path is shared with pill management.** History itself only writes on `toggleHistoryTaken`; the events log is written by the pill-management feature (see PRD-pill-management.md), History just displays it.

## Edge cases and empty states

- **No log document exists for a past date** (app wasn't opened that day): `dayScore` receives `log = null/undefined` from `buildActivePillsForDate` path, returns `{pct:0, taken:0, total:0}` — shows as 0% rather than "no data," worth confirming this reads correctly to the user rather than looking like a missed day with no medication.
- **A pill was deleted with no version history** (added and stopped same day, or deleted without ever being versioned): still shows correctly for days it was active, via either `pr:list` (if never archived) or `pr:archived` (if archived on delete — confirm this against pill-management's delete path, not fully traced here).
- **Events array grows unbounded**: no pruning logic found. Over a long usage period this becomes a large single Supabase value. Not currently a problem at expected usage volume, worth a note for later.
- **30-day window is fixed**: `getLast30Days()` always returns exactly 30 calendar days from today, no pagination for older history.

## Must match exactly vs UX can change

**Must match exactly:**
- `buildActivePillsForDate` logic, especially the archived-pill date-range inclusion. This is what keeps History accurate across pill versioning, losing this breaks the entire safety case for the versioning feature in PRD-pill-management.md.
- `dayScore` being purely derived, never stored, so it can't drift out of sync with the log.
- Events being append-only (prepend) and shared across add/edit/delete, so a rebuild's pill-management logic must write here too, not treat it as History-only.
- Retroactive toggle being a full read-modify-write per day, not a shared cached document, to avoid cross-day write collisions.

**UX can change:**
- 30-day window could become paginated or configurable
- Timeline presentation (integrated into day cards vs separate view)
- Toggle interaction pattern (tap vs swipe vs long-press)

## Out of scope

- Editing as-needed dose timestamps from within History (that's Today-only, see PRD-as-needed.md)
- Exporting History specifically as a standalone document (full backup export covers this, see Settings feature)
- Any retention/pruning policy for `pr:events`

## Open questions

- Does a 0% score on a day with no log document need a distinct "no data" state instead of looking identical to "took nothing"? Worth confirming with Claire on-device, this affects how doctors reading exported history would interpret gaps.
- Is there a cap or archival plan for `pr:events` as it grows over months/years of use?
