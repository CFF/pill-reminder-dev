# Posology — Product Requirements Document

## Product vision

Posology reduces the mental load of managing daily medications. It is the quiet, reliable companion that remembers so you don't have to — and speaks up when it matters for safety.

**Primary job:** Comfort — make it effortless to know what to take and when.  
**Secondary job:** Safety — prevent dangerous double-doses, especially for as-needed pain relief.  
**Tertiary job:** Compliance — keep a log accurate enough to show a doctor.

---

## Users

| User | Description |
|---|---|
| Primary | Claire — manages her own medications, power user, configures everything |
| Secondary | Lysiane — shared user, lighter usage, receives a pre-configured link |
| Future | Small open user base via personal URL tokens |

**Access model:** URL token (`?u=name-XXXXXXXXXX`), a high-entropy, non-guessable token that is the user's namespace. No password, no registration. Since the 2026-07 lockdown, the `data` table cannot be read with the public key — access is only through token-scoped database functions — so the medication and medical-context data can no longer be dumped from the table, and tokens can no longer be guessed. What is **not** protected: anyone who obtains a valid link has full access to that namespace (the link is the key). Removing that would require real per-user login, which is out of scope. First-name tokens are deprecated; issue only high-entropy ones.

---

## Feature registry

Each row is the canonical source of truth for that feature's behavioral contracts. If a feature file exists, it supersedes the summary below for that feature.

| # | Feature | File | Status |
|---|---|---|---|
| 1 | Scheduled sessions | — | Summary below only |
| 2 | As-needed medications | [PRD-as-needed.md](PRD-as-needed.md) | Documented |
| 3 | History | [PRD-history.md](PRD-history.md) | Documented |
| 4 | AI Recommendations (tips) | — | Summary below only |
| 5 | Nurse (AI chat) | [PRD-nurse.md](PRD-nurse.md), [PRD-medical-context.md](PRD-medical-context.md) | Documented — known bug: chat lost on reload |
| 6 | Settings | — | Summary below only |
| 7 | Open access (local-first) | [PRD-open-access.md](PRD-open-access.md) | Backlog, not scheduled |
| — | Pill management (add/edit/versioning/scan/autocomplete) | [PRD-pill-management.md](PRD-pill-management.md) | Documented |

---

## Features

### 1. Scheduled sessions (Morning / Lunch / Evening)

**What it does:** Groups daily pills by session, lets the user check them off.

**Use cases:**
- UC1: User opens app in the morning, taps each pill to mark it taken
- UC2: User missed lunch pills — history shows them as not taken
- UC3: A pill expires (durationDays reached) — auto-archived on load, shown in history

**Behavioral contracts:**
- A session is hidden if it contains zero active pills (no empty "No medicines" placeholder)
- Checking off a pill writes `{id: true}` to `pr:day:YYYY-MM-DD`
- Auto-archive runs on every load via `archiveExpired()` — does not require user action

**Out of scope:** Reminders / push notifications (open architectural question — local SW vs external service).

---

### 2. As-needed medications

**What it does:** Tracks timestamped doses for PRN medications with optional daily max and minimum interval between doses.

**Use cases:**
- UC1: User takes an ibuprofen, taps +. Card shows "1x today · last at HH:MM"
- UC2: User tries to take a second dose too soon. Button shows "Next at HH:MM", is visually blocked
- UC3: User reaches daily max. Card shows "Max 3x reached" warning, button disabled
- UC4: User took a dose at 23:54 but forgot to log it. Next morning, taps "Add missed dose", selects yesterday 23:54. Interval guard immediately reflects the correct "next dose" time
- UC5: User wants to correct a logged time. Taps the time in the detail sheet, edits inline

**Behavioral contracts — source of truth per element:**

| Element | Source | Scope |
|---|---|---|
| "Nx today" count | `todayOnlyTs(timestamps).length` | Calendar day |
| "last at HH:MM" | `todayTs[count-1]` | Calendar day |
| "Next at HH:MM" / blocked button | `timestamps[timestamps.length-1]` | 24h window |
| "Max Nx reached" | `count >= maxDailyDose` | Calendar day |
| Detail sheet dose list | `todayOnlyTs(timestamps)` | Calendar day |

**Invariant:** the 24h window in `asNeededTs` state exists solely for the interval guard across midnight. It is never displayed and never written to today's Supabase record.

**Missed dose (UC4) contract:**
- Picker: datetime-local, min = yesterday 00:00, max = now
- If date = today → normal write path
- If date = yesterday → write to `pr:asneeded:yesterday`, update state for interval guard, do NOT pollute today's Supabase record

---

### 3. History

**What it does:** Shows a scrollable log of past days with session-by-session breakdown.

**Use cases:**
- UC1: User reviews yesterday — sees which pills were taken and which weren't
- UC2: User needs to show doctor compliance over 30 days
- UC3: User spots a missed dose and wants to correct it retroactively (not yet implemented)

**Behavioral contracts:**
- Shows pills that were **active on that date**, using archived pill data — not just current pills
- Empty sessions are hidden (not shown as "No medicines")
- As-needed doses in history: shows timestamps from `pr:asneeded:YYYY-MM-DD`

---

### 4. AI Recommendations

**What it does:** Shows drug-specific dietary and care tips for active medications, generated by Claude via a Supabase Edge Function.

**Use cases:**
- UC1: User adds a new pill → tips regenerate automatically
- UC2: User sees "Do NOT exceed X mg/day" and adjusts their scheduled doses
- UC3: User opens app — tips load from Supabase cache if pill list hasn't changed

**Behavioral contracts:**
- Tips are cached by hash of the pill list — same pills = no API call
- "New" chip (blue) shown on unseen cards; seen state persisted in localStorage
- Separator between tip points: no absolute-positioned lines (plain flex rows)
- Disclaimer border: `var(--sep-strong)` not `var(--sep)`
- Tips cleared if pill list changes (forces regeneration)

---

### 5. Nurse (AI chat)

**What it does:** Conversational AI assistant aware of the user's medication context.

**Use cases:**
- UC1: User asks "can I take ibuprofen with pregabalin?" — gets a contextual answer
- UC2: User describes a symptom — Nurse gives general guidance, recommends consulting a doctor

**Behavioral contracts:**
- API calls route through Supabase Edge Function `nurse-chat` — never direct from client (CORS/CSP on GitHub Pages)
- Nurse header is `position:sticky` so it stays visible while scrolling the conversation
- Nurse is not a substitute for medical advice — responses always include appropriate caveats

---

### 6. Settings

**Use cases:**
- UC1: User exports JSON backup before reinstalling the PWA
- UC2: User imports JSON backup after reinstalling
- UC3: User fills in profile (age, weight) so Nurse has better context
- Backup schema v2 (2026-07): export/import covers `pr:list`, `pr:archived`, `pr:day:*`, `pr:asneeded:*`, `pr:profile`, `pr:medical-context`, `pr:events`. Any future Supabase key holding user data MUST be added to both export and import in the same commit that introduces it.

---

### 7. Open access via local-first use

**Status: Backlog** — not scheduled. See [PRD-open-access.md](PRD-open-access.md).

**What it does:** Let Claire hand the app to anyone, who then uses Posology fully on their own device with data stored locally (IndexedDB), no account or token. Sidesteps the cloud token's security risks for those users. Reverses the current "Supabase only" storage rule for this mode.

---

## What is explicitly out of scope (current phase)

- Push notifications / reminders (unresolved: local SW notifications vs external service)
- Multi-user admin (Claire managing Lysiane's pills for her)
- Caregiver view / shared household
- Clinical-grade compliance export (PDF, e-prescriptions)
- Drug interaction checking beyond AI tips

---

## Technical constraints that affect product decisions

- **Single `index.html`, no build step** — no npm packages at runtime, no code splitting. Features requiring heavy dependencies must route through Supabase Edge Functions.
- **GitHub Pages hosting** — no server-side rendering, no dynamic routes, no webhooks.
- **iOS PWA limitations** — no background push notifications from SW without a native wrapper. "Add to Home Screen" is the install path.
- **URL token access** — sharing the URL = sharing the account. Design features with this in mind: no private notes, no sensitive data beyond medication names.

---

## Change protocol

Any patch that touches the as-needed data flow (timestamps, `asNeededStatus`, `takeAsNeeded`, `updateTimestamps`, `addMissedDose`, the detail sheet, or the Today card display) must:

1. Identify which behavioral contract (table above) is affected
2. Audit every place the changed variable is used as an array index
3. Be tested on dev before prod
4. Include a one-line comment in the commit message naming the contract

This protocol exists because a change to `count` derivation in pr-v101 introduced a bug where `timestamps[count-1]` indexed into a differently-sized array, showing yesterday's "last at" time instead of today's.

---

## Future vision — Condition-based treatment tracking

**Concept:** Evolve Posology from a pill reminder into a personal treatment journal organized around medical conditions.

**Core idea:** A medication doesn't exist in isolation — it's prescribed for a reason. Allowing users to associate a drug (active or archived) with a condition (e.g. "Prednisone → Autoimmune flare", "Ibuprofen → Chronic back pain") transforms the history view from a compliance log into a readable treatment narrative.

**Use cases (not yet scoped):**
- UC1: User links a medication to a condition when adding or editing it
- UC2: History view can be filtered or grouped by condition
- UC3: Archived medications remain visible under their condition — so a user can show a doctor the full treatment timeline for a given condition, including past drugs and doses
- UC4: Nurse AI has condition context, enabling more relevant answers ("I'm managing X, currently on Y")

**Why this matters:** Users managing chronic or complex conditions need more than "did I take my pill today." They need a record that makes sense to a doctor — what was tried, at what dose, for which condition, and when it changed.

**Dependencies before this can be scoped:**
- Medication versioning must be solved first (dose change history, multi-intake-per-drug model)
- Condition entity needs definition: free text vs. ICD code lookup vs. user-defined tags

**Status:** Vision only. Not scheduled.
