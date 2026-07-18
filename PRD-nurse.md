# Feature PRD: Nurse (AI chat)

> Derived from: [PRD.md](PRD.md)
> Any principle not explicitly overridden here follows the product PRD.
> Context injection, medical context fields, and the banner are fully specified in [PRD-medical-context.md](PRD-medical-context.md) — this file does not repeat that content. This file covers the chat mechanism itself: sending, rendering, state, and error handling.
> Reverse-engineered from `index.html` at pr-v194 (dev).

## Overview

The conversational tab where the user asks free-form questions about their medications. Built as a single `HealthAssistant` component whose message state (`msgs`) lives in `App`, not in the component itself, so a conversation survives switching tabs and coming back.

## Problem this feature solves

Tips are static and generated once per pill-list change. Sometimes the user has a specific question ("can I take this with ibuprofen," "what if I miss a dose") that a pre-generated tip card can't anticipate. Nurse gives a live, contextual answer without the user re-explaining their situation each time, per PRD-medical-context.md.

## Users

Same as product PRD. No feature-specific segment.

## Acceptance criteria

- [ ] Sending a message shows it immediately, then a loading state, then the reply
- [ ] Switching to another tab and back preserves the conversation (no state loss)
- [ ] A failed request shows a retry option, not a lost message
- [ ] Suggestion chips send their text as if typed, and disappear after first use is implied by normal chat flow
- [ ] The header stays visible while scrolling a long conversation (`position:sticky`)
- [ ] A "scroll to bottom" affordance appears when the user has scrolled up during an active or recent conversation
- [ ] The system prompt correctly omits any field not filled in, per PRD-medical-context.md's injection rules

## Functional requirements

### Sending a message (`send` / `runApi`)

- `send(text)` takes either typed input or a suggestion chip's text, ignored if empty or already loading
- Appends the user message to `msgs` immediately (optimistic), then calls `runApi`
- `runApi` builds the system prompt fresh on every call via `buildSystemPrompt()` (not cached, always current pill list and context), and calls `nurse-chat` with `{system, messages}`
- **Message merging (`apiSafe`):** before sending, consecutive messages with the same role are merged into one (keeps the last). This exists because the Anthropic API requires strictly alternating user/assistant turns, and the app's local state doesn't always guarantee that shape.
- On success, appends `{role:"assistant", content: reply}` to `msgs`. If the response has no usable text, falls back to a generic "I couldn't process that" message rather than showing nothing.
- On failure (non-OK response or network error), sets an error state and does not append a broken assistant message — the retry button re-sends the same `msgs` array via `runApi(msgs)`.

### Suggestion chips

Current set: `"Side effects?"`, `"Foods to avoid?"`, `"Drug interactions?"`. Tapping one calls `send()` with that text, same path as typed input. (Note: DESIGN.md known-debt item #5 flagged a prior chip, "Can I stop early?", as steering toward a forbidden question — that chip has since been removed/replaced in the current set. Design debt item can be marked resolved.)

### Scroll behavior

- Auto-scrolls to the newest message on new content, but skips the very first render (`didScroll` ref guard) so opening the tab doesn't jank-scroll an already-loaded conversation
- An `IntersectionObserver` on the bottom anchor toggles a "scroll to bottom" button when the user has scrolled the last message out of view

### Keyboard handling (iOS)

Uses `window.visualViewport` resize/scroll listeners to compute keyboard height offset (`kb` state) and adjusts layout accordingly — necessary on iOS Safari where the keyboard doesn't reliably resize the layout viewport on its own.

## Data reads and writes

**No direct Supabase writes from this feature.** Chat messages (`msgs`) are session-only state held in `App`, not persisted. Closing the app or reloading loses the conversation. Medical context and profile are read (not written) here, see PRD-medical-context.md for their storage.

**Edge Function:** `nurse-chat`, called with `{system: systemPrompt, messages: apiSafe(msgs)}`. Same endpoint used by prescription scan (see PRD-pill-management.md) but with a real system prompt instead of an empty one.

## Edge cases and empty states

- **Empty pill list:** `buildSystemPrompt` presumably omits the "ACTIVE TREATMENT" block per PRD-medical-context.md's injection rules — not re-verified here, inherits that contract.
- **Two consecutive user messages in state** (shouldn't normally happen given the UI, but defensive): `apiSafe` merges them before sending rather than erroring.
- **Request fails:** error is shown, no message is silently dropped, retry re-sends the exact same conversation state.
- **No response text in a successful API response:** shows a fallback string rather than an empty bubble.

## Must match exactly vs UX can change

**Must match exactly:**
- `msgs` state living in `App`, not the chat component, so tab switches don't lose the conversation
- The `apiSafe` role-merging behavior, required by the Anthropic API's alternating-turn constraint
- System prompt being rebuilt fresh on every send (never cached/stale)
- Error handling that preserves the user's message and conversation state on failure, only the assistant's response is retryable

**UX can change:**
- Sticky header, scroll-to-bottom button styling, suggestion chip wording and count
- Keyboard-offset handling approach, as long as the input stays usable when the iOS keyboard is open
- Loading state presentation (currently generic per DESIGN.md debt item #3, "most generic chatbot archetype," queued to warm up)

## Known bug — chat does not survive reload

Expected behavior: conversation history should persist as long as the app is installed, same as any other data in the app.

Actual behavior: `nurseMsgs` is `useState([])` in `App`, never persisted to Supabase or localStorage. It survives tab switches only because `App` never unmounts during normal navigation. A PWA reload, relaunch from the home screen icon after being backgrounded long enough to be killed, or a hard refresh loses the entire conversation.

Fix direction: persist `nurseMsgs` to a new Supabase key (e.g. `pr:nurse-history`) on every update, load it in `loadData()` alongside the other keys. Not implemented here, this file documents the gap; the fix belongs in the bug-fix track, not the documentation pass.

## Out of scope

- Multi-turn context beyond what's in the current session's `msgs`
- Any Nurse action beyond text response (no ability to add/edit pills from chat, unlike scan which is a separate entry point)

## Open questions

- DESIGN.md debt items #3 (generic chatbot look) and #4 (disclaimer repeated three times) are still open — not addressed by this documentation pass, flagged for a design session.
