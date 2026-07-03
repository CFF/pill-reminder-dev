# Posology — Security & Stability Audit (2026-07-02)

> **LIFECYCLE OF THIS FILE — READ FIRST**
> This file is a snapshot, not a living document. It is the execution spec for the tasks below.
> When the **last open task closes**, the closing agent must: (1) verify every "Exit obligation" below has been written into its target doc, (2) run the Closing Checklist at the bottom, (3) **delete this file in that same commit**. Do not leave AUDIT.md in the repo after execution: a stale audit describing fixed vulnerabilities is worse than no audit.

**Audited state:** dev `pr-v182` (157,681 bytes), prod `pr-v131` (155,777 bytes). JSX valid, version count == 2, no IndexedDB regressions, dev/prod divergence ~50 lines (Nurse banner work, scan debug logs, dev title/splash).

**Coordinator:** Claude (Fable 5), this audit chat. Each task below is executed in its **own dedicated chat** by the assigned model. The executing agent must read this file, CLAUDE.md, and the relevant target file fresh from GitHub before acting.

---

## Rules that bind every task (no exceptions)

These restate the CLAUDE.md gates. They are repeated here because the executing agent may be a different model with no memory of past incidents.

1. **Dev first.** All `index.html` changes go to `cff/pill-reminder-dev`, are validated on Claire's iPhone, then ported surgically to `cff/pill-reminder`. Never copy dev wholesale to prod.
2. **Fresh fetch.** Fetch file + SHA via GitHub Contents API in the same step as the patch. Fetch a fresh SHA again immediately before PUT. Never reuse anything cached earlier in the session.
3. **Preview before commit.** Show the diff (or UI preview for visual changes) and wait for Claire's explicit approval ("Go", "Ok") before any PUT.
4. **JSX validation.** Concatenate all `<script type="text/babel">` blocks, parse with `@babel/parser` (`plugins:['jsx'], sourceType:'script'`). Abort on failure.
5. **Version bump.** Bump `pr-vNNN`, assert count == 2 before and after.
6. **String patches assert count == 1** on the anchor before replacing.
7. **Python, not bash heredocs**, for reading/patching/encoding (file too large for heredocs).
8. **Edge Functions cannot be deployed by Claude.** Supabase is unreachable from Claude's environment. The agent prepares the full function source, Claire pastes it into the Supabase dashboard, then validates on device. The repo copy under `supabase/functions/` (prod repo) must be updated in the same task so the backup matches what is live.
9. **Testing always uses `?u=test`**, never `?u=claire`.
10. **GitHub token:** read silently from Project Instructions. Never print, echo, or mention it. Write it once to a temp file and reference the file in scripts.

---

## TASK 1 — Harden `nurse-chat` and `get-tips` Edge Functions
**Priority:** P0 · **Model:** Opus 4.8 · **Files:** `supabase/functions/nurse-chat/index.ts`, `supabase/functions/get-tips/index.ts` (prod repo) + live paste into Supabase dashboard

**Problem (user impact):** `nurse-chat` accepts any `system` and `messages` from anyone holding the anon key, which is public in the HTML. Anyone who views source can spend Claire's Anthropic credits without limit, including expensive image calls. `get-tips` shares the open CORS / no rate limit exposure, at lower severity.

**Required changes, minimum scope:**
- `nurse-chat`: wrap body parsing and upstream fetch in try/catch (a malformed request currently returns a raw 500)
- `nurse-chat`: guard for missing `ANTHROPIC_API_KEY` before calling Anthropic (mirror the guard already in `get-tips`)
- `nurse-chat`: **cap `max_tokens` server-side** (ignore/clamp any client value; current hardcoded 600 is fine as the cap)
- `nurse-chat`: **prepend a server-side system prefix** that scopes the function to Posology use (medication/health assistant + pharmacy label reading). Client-supplied `system` is appended after the prefix, never replaces it. The prefix must not break the two existing callers: Nurse chat (sends `buildSystemPrompt()` output) and pill scan (sends `system:""` + image)
- `nurse-chat`: cap request size (reject bodies over a sane limit, e.g. ~5 MB, to bound image abuse)
- Both: restrict `Access-Control-Allow-Origin` to `https://cff.github.io` (verify the pill-scan and Nurse flows still work from both prod and dev pages — same origin, different paths, so one origin value covers both)
- Both: basic rate limiting is **desirable but optional** in this pass. If Opus judges it low-risk (e.g. a per-IP counter in a Supabase table), propose it; otherwise log it as a follow-up. Do not let it block the P0 fixes.

**Acceptance criteria (on Claire's iPhone, `?u=test` on dev):**
- Nurse chat answers normally
- Pill scan from camera/library still parses a label
- AI tips still generate after a pill list change
- A malformed request (Opus provides Claire a one-line curl or explains how it was verified) returns a clean JSON error, not a raw 500

**Exit obligation → SPECS.md**, replace the "Recommended hardening (not yet done)" block with, verbatim:
> **Hardening in place (audit 2026-07):** both functions guard for a missing `ANTHROPIC_API_KEY`, wrap parsing and the upstream fetch in error handling, and restrict CORS to `https://cff.github.io`. `nurse-chat` additionally clamps `max_tokens` server-side, prepends a locked system prefix before any client-supplied `system`, and caps request body size. Any future edit to these functions must preserve all of the above.

---

## TASK 2 — Fix silent write failure in `sbSetRemote`
**Priority:** P0 · **Model:** Sonnet 4.6 · **File:** `index.html` (dev, then prod)

**Problem (user impact):** `sbSetRemote` does `await fetch(...)` and never checks `res.ok`. A Supabase HTTP error (4xx/5xx) does not throw, so `sbSet` believes the write succeeded, removes the entry from the offline sync queue, and the dose log is **silently lost**. This is the same failure family as the v57 data loss.

**Patch, exact scope:** in `sbSetRemote`, capture the fetch result and throw on non-OK:
```js
async function sbSetRemote(key,value){
  const res=await fetch(`${SUPA_URL}/rest/v1/data`,{ ...unchanged... });
  if(!res.ok)throw new Error("sbSet HTTP "+res.status);
}
```
The existing `try/catch` in `sbSet` and `flushSyncQueue` then routes the failed write into the sync queue, which is the correct behavior. **No other call site changes.** Anchor on the current one-line body of `sbSetRemote`; assert count == 1.

**Acceptance criteria:** app loads, a dose check-off persists after reload (`?u=test`). Behavior is otherwise invisible; the fix is defensive.

**Exit obligation → CLAUDE.md**, add under "Known gotchas", verbatim:
> - **Every Supabase fetch must check `res.ok`.** `fetch` only rejects on network failure, not HTTP errors. A write path that ignores `res.ok` reports success on failure and drops data silently (root cause class of the v57 loss; fixed in `sbSetRemote` during the 2026-07 audit). Any new fetch to Supabase must follow this rule.

---

## TASK 3 — Add an ErrorBoundary
**Priority:** P1 · **Model:** Sonnet 4.6 · **File:** `index.html` (dev, then prod)

**Problem (user impact):** there is no ErrorBoundary and no global error handler. Any render exception is a blank white screen with no way out except reinstalling or clearing Safari (the v81 incident). For a daily medication app, a blank screen at pill time is a safety failure, not a cosmetic one.

**Patch, exact scope:**
- One class component `ErrorBoundary` (class is required; hooks cannot catch render errors), defined **before `App`** like all other components (Babel constraint)
- `componentDidCatch` renders a minimal fallback styled with existing tokens: centered, `--bg` background, icon, "Something went wrong" + French subtitle, and a full-width accent button "Reload" doing `window.location.reload()`
- Wrap the root render: `<ErrorBoundary><App/></ErrorBoundary>`
- No logging service, no error reporting. Keep it under ~30 lines.

**Acceptance criteria:** app renders normally on `?u=test`. (Sonnet may verify the boundary by temporarily throwing in a component during preview, but the committed code must not contain the test throw.)

**Exit obligation → SPECS.md**, add one line under Architecture, verbatim:
> - Root render is wrapped in `ErrorBoundary` (class component, defined before `App`) — render crashes show a reload fallback instead of a blank page

---

## TASK 4 — Complete the backup (export + import)
**Priority:** P1 · **Model:** Sonnet 4.6 · **File:** `index.html` (dev, then prod)

**Problem (user impact):** the JSON export contains `pillList, archived, logs (pr:day:*), profile` only. It **omits** `pr:asneeded:*` (as-needed dose history), `pr:medical-context`, and `pr:events`. A user who reinstalls and restores silently loses their pain-medication history and medical context. The backup promise in Settings is currently broken without anyone knowing.

**Patch, exact scope:**
- Export: also gather `pr:asneeded:*` (via `sbGetAllDayKeys("pr:asneeded:")`), `pr:medical-context`, `pr:events`; add them to the exported object; add a `schema: 2` field
- Import: restore all keys present in the file; must accept **both** schema 1 (old backups, missing keys skipped gracefully) and schema 2
- **Do not touch** any as-needed runtime logic (`asNeededTs`, `takeAsNeeded`, `asNeededStatus`). This task writes/reads Supabase keys only. If the agent finds itself editing those functions, stop: the scope is wrong.

**Acceptance criteria (on `?u=test`):** export a backup, wipe test data (or use a second token like `?u=test2`), import, verify: pills, today's log, as-needed history visible in History, medical context fields repopulated in Settings.

**Exit obligation → PRD.md**, in Settings use cases, verbatim:
> - Backup schema v2 (2026-07): export/import covers `pr:list`, `pr:archived`, `pr:day:*`, `pr:asneeded:*`, `pr:profile`, `pr:medical-context`, `pr:events`. Any future Supabase key holding user data MUST be added to both export and import in the same commit that introduces it.

---

## TASK 5 — Token migration design (guessable `?u=` tokens)
**Priority:** P1 · **Model:** Opus 4.8 · **Type:** design first, then a separate approved implementation

**Problem (user impact):** tokens are first names. Anyone who reads the public source and guesses `?u=lysiane` can read **and modify** her medications, allergies, and medical history. The original PRD tradeoff ("no sensitive data beyond medication names") was broken the day the medical context feature shipped, so the access model must be upgraded to match the data it now protects.

**The design must answer, at minimum:**
1. Token format (e.g. `name-XXXXXX` with high-entropy suffix) and how new tokens are issued
2. **Migration path for existing users** — Claire and Lysiane must not lose data and must not need to do anything harder than tapping a new link once (Lysiane is non-technical and privacy-cautious). Likely shape: copy data to the new token server-side or on first load, old token redirects or is drained then deleted
3. What happens to the old rows (delete after grace period?)
4. RLS verification checklist Claire runs in the Supabase dashboard (Claude cannot): confirm UNIQUE `(user_token, key)`, confirm anon role cannot `select` without a `user_token` filter (blocking full-table dumps), state plainly what RLS **cannot** protect (a known token grants full access — that stays true)
5. Whether Lysiane's discussed local-only/IndexedDB option interacts with this (flag only, do not scope it here)

**Deliverable:** a written design reviewed by Claire in that chat. Implementation is a **separate task and commit**, spec'd by the design, dev first as always.

**Exit obligation → PRD.md**, update the Access model section to describe the new token format and replace the sentence "not appropriate for sensitive clinical data" with an accurate statement of what is and isn't protected after migration.

---

## TASK 6 — Housekeeping (fold into the next natural commit, no dedicated chat needed)
**Priority:** P2 · **Model:** Sonnet 4.6
- Strip the three scan debug `console.log`s (`SCAN RAW TEXT`, `SCAN PARSED`, `SCAN ENTRIES COUNT`) from dev **before** the Nurse banner work is ported to prod; they print label contents to the console
- Mirror the updated Edge Function sources into the dev repo too, or add one line to SPECS.md stating the backup lives in the prod repo only (pick one; today it is implicit)

---

## Closing Checklist (run by the agent closing the last task, with Claire)

1. Verify all five Exit obligations are present verbatim in CLAUDE.md / SPECS.md / PRD.md
2. Verify prod and dev both carry Tasks 2, 3, 4 (systematic diff, per CLAUDE.md)
3. Verify live Edge Functions on Supabase match the repo copies
4. **Delete AUDIT.md in this same commit**
5. **Token rotation (Claire, manual, last step):** at github.com/settings/tokens, revoke the current PAT and create a fine-grained PAT scoped to only `cff/pill-reminder` and `cff/pill-reminder-dev`, Contents: Read and Write, with an expiration date. Update Project Instructions immediately. Do this only after all tasks are done, so no agent chat breaks mid-work.

*Audit performed 2026-07-02 by Claude (Fable 5) on fresh fetches of both repos. Findings verified against source, not memory.*
