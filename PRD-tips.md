# Feature PRD: AI Recommendations (tips)

> Derived from: [PRD.md](PRD.md)
> Any principle not explicitly overridden here follows the product PRD.
> Reverse-engineered from `index.html` at pr-v194 (dev).

## Overview

Drug-specific dietary and care tips for active medications, generated once per unique pill list and cached, so the same pill list never triggers a repeat API call. Shown as cards with a "New" badge for tips the user hasn't scrolled past yet.

## Problem this feature solves

Reading a drug's full package insert is tedious and most of it doesn't apply. Tips surface the handful of things that actually matter for the user's specific combination of drugs (dietary conflicts, timing, max doses) without a repeat API cost every time the app opens.

## Users

Same as product PRD. No feature-specific segment.

## Acceptance criteria

- [ ] Adding, editing, or deleting a pill regenerates tips for the new pill list
- [ ] Same pill list on a later visit loads from cache, no API call
- [ ] Unseen tip cards show a "New" (blue) badge, badge clears when the card is scrolled into view
- [ ] If the API call fails, previously loaded tips remain visible rather than disappearing
- [ ] Removing a pill and reopening the app never shows a tip that references the removed pill
- [ ] Force-refresh regenerates tips even if a cache entry exists

## Functional requirements

### Cache key (`hashKey`)

Built from a plain-text description of every active pill (name, dose, as-needed flag, max daily dose), base64-encoded: `"pr:tips:" + btoa(unescape(encodeURIComponent(pillsDesc)))`. Any change to name, dose, or these specific fields produces a different key, any unrelated field change (schedule timing, reason, shape) does not, so cache hits are broader than "the exact same pill list."

### Load flow (on mount / pill-list change)

1. Build `pillsDesc` and `hashKey` from the current active pill list (morning + lunch + evening + as_needed)
2. If pill list is empty, set tips to `[]` immediately, no API call
3. Try `sbGet(hashKey)`. On a hit:
   - **Filter the cached tips against currently active pill names** before displaying: a tip is kept if its label is `"interactions"` (always kept) or if its label matches an active pill name (substring match, either direction). This is the safeguard against showing a tip for a pill that's since been removed, even though the cache key itself wouldn't have changed for an unrelated pill's removal in some edit paths.
   - If filtering removed anything, the filtered (smaller) list is written back to the same cache key, then displayed.
   - Compute new/unseen labels against `pr:seenTipLabels` (localStorage), set the "New" badge state accordingly, **then return** — no API call happens on a cache hit.
4. On a cache miss (or `sbGet` throwing): call `get-tips` Edge Function with `{pillsDesc}`, on success cache the result under `hashKey` and compute new/unseen labels the same way.

### `forceRefreshTips()`

A separate function, same pill-description construction, but skips the cache read entirely and always calls the Edge Function, then overwrites the cache entry. Triggered manually (regenerate button) rather than on every pill-list mutation.

### Error handling

The API-call `catch` block only logs to console (`console.error("Tips error", e)`), it does **not** clear `aiTips`. Whatever was loaded before (from cache or a previous successful call) stays on screen. This was a deliberate partial fix (previously the catch block called `setAiTips([])`, wiping recommendations on any transient failure).

### "New" badge mechanics

- `newTipLabels` (a `Set`) is initialized once from `localStorage.getItem("pr:newTipLabels")` on mount. This key is **never written anywhere in the app** (`setItem("pr:newTipLabels"` has zero matches). The initializer's output is also unconditionally overwritten within the same load cycle: every load path (cache hit or miss) ends by calling `setNewTipLabels(new Set(newLabels))` computed fresh from `pr:seenTipLabels`, before the user ever sees the initializer's value. Confirmed dead code, scheduled for removal in the next bug-fix pass — a 1-line deletion, zero behavior change since the read value is never used.
- The actual seen-state source of truth is `pr:seenTipLabels` (localStorage array of label strings), written by `markSeen(label)` and `markAllSeen()`. This is a distinct concept from `pr:newTipLabels`, not a duplicate of it: `pr:seenTipLabels` persists which labels the user has scrolled past, while `pr:newTipLabels` would have been a cache of which labels are currently flagged new. They're related but not interchangeable, the dead key just never got wired up before `pr:seenTipLabels` made it redundant in practice.
- `markSeen` is called via an `IntersectionObserver` per tip card (threshold 0.5), so scrolling a card into view for a moment marks it seen and clears its badge.
- `hasNewTips` (boolean, drives a tab-level indicator) is true whenever `newTipLabels.size > 0`.

## Data reads and writes

**Supabase keys:**
- `pr:tips:<base64-hash>` — one entry per unique pill-description hash, written on every successful API call or cache-filter correction, read on every load/pill-change

**localStorage:**
- `pr:seenTipLabels` — array of tip labels the user has scrolled past, source of truth for the "New" badge
- `pr:newTipLabels` — read once on mount, confirmed dead code (see Functional requirements above), never written, its value is unconditionally overwritten before use. Scheduled for removal, not a real storage contract.

**Edge Function:**
- `get-tips` — `POST {pillsDesc}`, returns a JSON array of `{label, ...}` tip objects, cached by hash on the client side, not server side

## Edge cases and empty states

- **Empty pill list:** tips set to `[]` immediately, no network call, no loading state shown.
- **API returns a non-array:** the `Array.isArray(tips)` guard means nothing is set and nothing is cached, `aiTips` simply keeps its previous value (same graceful-degradation behavior as an outright fetch error).
- **Cache hit but every tip gets filtered out** (all referenced pills removed): displays an empty tips list, and the emptied cache entry is written back, so a later reload of the same now-invalid hash won't re-fetch, it'll correctly show empty. This is only reachable if the hash key itself didn't already change, which requires an edit path that changes pill list membership without changing name/dose (edge case, not fully traced against pill-management's write paths).
- **`sbGet` throws** (network issue mid-load): falls through to attempting an Edge Function call directly, same as a clean cache miss.

## Must match exactly vs UX can change

**Must match exactly:**
- The hash key construction, since it determines cache reuse. A rebuild using a different key shape will get 100% cache misses on first load for existing users, which isn't wrong but does mean an unnecessary API call storm on migration day, worth being aware of when planning a rebuild's launch.
- The post-cache-hit filtering against active pill names, this is the primary defense against stale/irrelevant tips.
- Error handling never clearing previously-shown tips on failure.
- `pr:seenTipLabels` as the actual seen-state source, not `pr:newTipLabels`.

**UX can change:**
- Card layout, badge styling and placement
- Whether "New" clears on scroll-into-view vs on tap vs on a timer
- Force-refresh button placement and copy

## Out of scope

- Server-side caching or deduplication in the `get-tips` function itself, all caching is client-driven via the Supabase KV table
- Any per-drug partial-failure handling inside `get-tips` (flagged in SPECS.md as a pending Edge Function hardening item, separate from this client-side contract)

## Open questions

- The edge case where cache-key membership can theoretically diverge from pill-list membership (see Edge cases) wasn't fully traced against every pill-management write path. Worth a targeted check if a "phantom tip" bug is ever reported.
