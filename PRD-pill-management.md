# Feature PRD: Pill management (add, edit, versioning, scan, autocomplete)

> Derived from: [PRD.md](PRD.md)
> Any principle not explicitly overridden here follows the product PRD.
> Reverse-engineered from `index.html` at pr-v194 (dev). This feature was entirely undocumented before this file.

## Overview

Everything involved in getting a pill into the app and keeping its definition correct over time: manual entry, drug-name autocomplete, photo scan of a pharmacy label, editing an existing pill, and the versioning logic that decides whether an edit overwrites history or starts a new treatment record. This is the most structurally complex part of the app outside of as-needed dosing, and previously had zero documentation.

## Problem this feature solves

A pill isn't a static object, its dose, schedule, and duration change over a treatment's life (dose titration, doctor adjusts a prescription, refill with a different pack size). If every edit just overwrote the pill in place, History would silently rewrite the past — a dose shown as "taken" last week would retroactively show today's dose amount. This feature exists to keep History accurate while still letting cosmetic corrections (typo in the name, updated reason) happen without cluttering the record with fake new treatments.

## Users

Same as product PRD. No feature-specific segment.

## Acceptance criteria

- [ ] Adding a pill manually with name, dose, schedule, and dates works end to end
- [ ] Typing a drug name shows autocomplete suggestions from openFDA within ~2 keystrokes minimum
- [ ] Scanning a single-schedule label pre-fills the add form correctly (name, dose, session, duration)
- [ ] Scanning a multi-intake label (e.g. "2x per day") creates that many separate pill entries automatically, not one
- [ ] Editing only cosmetic fields (name, reason) overwrites the pill in place, no new ID, History unaffected
- [ ] Editing dose, schedule, start date, or duration triggers the version-change dialog before saving
- [ ] Confirming a version change archives the old pill with today as `endDate`, and creates a new pill with a new ID and `startDate` = today
- [ ] Today's log correctly drops the old pill ID and adds the new pill ID as untaken, without duplicating entries
- [ ] Tips regenerate after any pill add, edit, or delete (per existing tips contract)

## Functional requirements

### Manual add

Standard form: name, dose amount/unit, shape, one or more schedule entries (session + timing), start date, duration or "ongoing," optional reason, optional inventory count, optional as-needed fields (`maxDailyDose`, `minIntervalHours`).

### Drug name autocomplete (`searchDrugs`)

- Queries `api.fda.gov/drug/label.json`, matching `openfda.brand_name` and `openfda.generic_name` with a wildcard prefix match
- Minimum 2 characters before querying
- Returns up to 8 deduplicated results, brand names shown with their generic as subtitle and vice versa
- Silent failure: any fetch error or non-OK response returns an empty array, no error shown to the user
- **This is the only feature depending on an external, non-Supabase, non-Anthropic API.** If openFDA is down or rate-limits, autocomplete just stops offering suggestions; manual typing still works.

### Photo scan (`handleScanLabel`)

- User picks or photographs a label image
- Image is base64-encoded client-side and sent directly to the `nurse-chat` Edge Function with a `type:"image"` content block plus a structured extraction prompt (see Data reads/writes)
- The prompt requests a JSON array (never a single object), one element per daily intake time, with fields: `name`, `dosageAmt`, `dosageUnit`, `quantity`, `session` (normalized to morning/lunch/evening, handles French labels: matin/dejeuner→morning, midi/diner→lunch, soir/coucher→evening), `timing`, `durationDays` (months converted to days), `ongoing`, optional `reason` (max 5 words, same language as label)
- Response text is stripped of markdown code fences before `JSON.parse`
- **Single-intake result:** pre-fills the add form fields directly, user reviews and saves manually
- **Multi-intake result:** each array entry is saved as its own pill immediately via `onSave()`, no per-entry review step. Entries without a `name` are silently skipped.

### Edit and the clinical-vs-cosmetic split

On save, the app diffs the edited fields against the original pill:

```
clinicalChanged = dosageAmt changed
                OR dosageUnit changed
                OR dose changed
                OR startDate changed
                OR durationDays changed
                OR schedules changed (session+timing, order-independent via JSON compare)
```

- **If nothing clinical changed** (only name, reason, shape, or session reassignment without a schedule change): the pill is updated in place, same ID. If the primary session changed, the pill moves between session buckets in `pr:list`; today's log is patched so sessions the pill left are removed and newly added sessions get a fresh `false` entry (skips `as_needed`).
- **If something clinical changed:** the edit is held in `pendingVersionEdit` state and a confirmation dialog opens instead of saving immediately. The dialog optionally asks for a new inventory count.

### Version confirmation flow

On confirm, four things happen in sequence:
1. **Archive the old version.** Original pill object gets `endDate: today` and `archivedOn: today`, appended to `pr:archived` under its session.
2. **Create the new version.** New pill gets a new ID (`session[0] + Date.now()`), the edited fields, `startDate: today`. If an inventory count was entered, `durationDays` is recalculated as `floor(inventoryCount / dosesPerDay)` where `dosesPerDay` = number of non-as-needed sessions.
3. **Swap in `pr:list`.** Old ID removed from its session bucket, new pill added to its (possibly different) session bucket.
4. **Patch today's log.** Old ID removed from all sessions, new ID added as untaken to its session(s).

This is the mechanism that keeps History accurate: a dose logged yesterday still points to yesterday's pill ID and yesterday's archived dose amount, even after today's dose changes.

## Data reads and writes

**Supabase keys:**
- `pr:list` — read and written on every add/edit/delete
- `pr:archived` — written when a clinical edit is confirmed (old version) and when a pill naturally expires (`archiveExpired()`, per scheduled-sessions feature)
- `pr:day:YYYY-MM-DD` (today only) — patched on session changes and on version swaps

**External API (non-Supabase):**
- `api.fda.gov/drug/label.json` — autocomplete, no auth, public API, read-only

**Edge Function:**
- `nurse-chat` — reused for label scanning (Vision-capable, accepts `{system, messages}`, image content block). Not a separate function; scan and chat share the same endpoint.

## Edge cases and empty states

- **Scan produces invalid JSON:** `JSON.parse` will throw. Not wrapped in a visible user-facing error message in the excerpt reviewed — worth confirming behavior on-device (loading state may hang or fail silently).
- **Scan finds zero valid entries:** multi-entry loop skips entries without `name`; if all entries lack a name, nothing is added and no pill appears, with no explicit "scan failed" message shown to the user.
- **Autocomplete with < 2 characters:** returns immediately, no API call.
- **Editing a pill with no `schedules` array (legacy shape):** falls back to `[{session: currentSession, timing: ""}]` before diffing, so old pills without a `schedules` field don't false-trigger a clinical change.
- **Version edit with no inventory count entered:** `durationDays` falls back to the edited value the user typed, inventory fields are omitted from the new pill.

## Must match exactly vs UX can change

**Must match exactly:**
- The clinical vs cosmetic diff logic exactly as listed above — this is what protects History integrity. A rebuild that treats all edits as "cosmetic" (overwrite in place) will retroactively corrupt past compliance data.
- The 4-step version confirmation sequence (archive old → create new with new ID → swap in list → patch today's log), including the new-ID generation scheme, since other features (as-needed, tips cache) key off pill ID.
- Multi-intake scan producing N separate pills, not one pill with N schedule entries.
- The session-normalization mapping for French label terms, if scan is rebuilt with a different model/prompt.

**UX can change:**
- Add form layout, scan button placement, autocomplete dropdown styling
- Version confirmation dialog presentation (currently a fixed-position sheet)
- Whether scan pre-fills-then-review (single) vs auto-saves (multi) — this is a product behavior worth revisiting, not a technical constraint, see Open questions

## Out of scope

- Editing an archived (past) pill version's historical record — archived entries are read-only once created
- Batch edit of multiple pills at once
- Undo for a confirmed version change (would require manually deleting the new pill and un-archiving the old one)

## Open questions

- Should multi-intake scan results get a review step before saving, matching the single-intake flow, instead of auto-saving immediately? Currently inconsistent between the two paths, worth a product decision rather than treating as a bug.
- No visible error handling if the scan API call fails or returns malformed JSON. Worth confirming actual on-device behavior before deciding if this needs a fix.
