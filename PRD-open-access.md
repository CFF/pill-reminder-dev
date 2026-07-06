# Feature PRD: Open Access via Local-First Use

> Derived from: [PRD.md](PRD.md)
> Any principle not explicitly overridden here follows the product PRD.
> **Status: Backlog** — not scheduled. This document captures the approach and the decision space so the work is ready to scope when prioritized.

## Overview

Posology stays Claire's tool. This feature lets Claire hand the app to *anyone* — no account, no setup, no link she has to generate — and that person uses Posology fully on their own device, with their data stored locally and never sent to the cloud. Opening up this way sidesteps the security problems that come with shared cloud accounts, because a local user's data never leaves their phone.

## Problem this feature solves

The product's cloud access model (`?u=name`) is fine for Claire but does not safely scale to people she doesn't set up: tokens are guessable, and everything lives in one shared database. Rather than harden accounts for strangers, the local-first approach removes the problem: if a new user's data stays on their device, there is no token to guess and no shared table to dump. Anyone can use Posology, privately, by default.

## The approach (decided)

Local-first storage. A person who opens Posology without being one of Claire's cloud users runs entirely on-device: their pills, logs, history, and medical context are stored in the browser's local database (IndexedDB), not in Supabase. No account, no token, no registration.

This is the build direction, not an open fork. What remains open is how Claire herself and any future sync fit around it (see Open questions).

## Users

- **Claire (main user)** — keeps her current setup so she doesn't lose cross-device access. Whether she also has a local option is an open question below.
- **Anyone (local user)** — opens the app and just uses it, data on-device only. No account, no link from Claire required. Fully private by construction.

## Functional requirements

- A person can use the full app — tracking, history, as-needed, recommendations, Nurse — with all personal data stored **locally on their device**, no cloud account.
- On first open (no cloud token present), the app works locally by default rather than blocking on a token-entry screen.
- Local data never leaves the device except when the user themselves exports a backup.
- Backup export/import works for local users, so they can move to a new phone (there is no cloud sync to do it for them). The audit's completed backup work supports this.
- AI features (`nurse-chat`, `get-tips`) still route through the shared Edge Functions, so they must enforce per-user rate limits and a cost ceiling before the app is handed out widely — a local user still spends the shared Anthropic budget when they use the AI.

## Non-functional requirements

- **Privacy by construction:** a local user's medical data is stored only on their device. This is the core promise and must be true, not aspirational.
- **No new cloud attack surface for local users:** because they have no cloud namespace, the token-guessing and table-dump risks simply do not apply to them.
- **Fits the technical constraints:** single `index.html`, no build step, GitHub Pages, iOS PWA. Note this reverses the current "Supabase only, never IndexedDB" rule in SPECS.md, so that rule would need a documented exception scoped to this mode.

## Relationship to the token-security work

The token-security design (2026-07 audit) still matters, but only for **cloud users** — Claire, and anyone who later opts into cloud sync. Local users are outside its scope entirely. The two efforts are complementary: local-first protects the many; token security protects the few who sync.

## Acceptance criteria (high-level, to refine when scheduled)

- [ ] A brand-new user can open the app and use every feature with no account and no token, data stored on-device.
- [ ] A local user's data is verifiably never written to Supabase.
- [ ] A local user can export and re-import their data to move devices.
- [ ] AI usage by a local user cannot run up an unbounded shared bill.
- [ ] Claire's existing cloud experience is unchanged (no data loss, still cross-device).

## Out of scope

- Accounts, passwords, or login of any kind.
- Cross-device sync for local users (the deliberate trade-off of staying local).
- Caregiver / household management.
- Changing the medication-tracking behavior itself.

## Open questions

- **Does Claire also get a local option, or does she stay cloud-only?** Owner: Claire.
- **Should a local user be able to opt into cloud sync later** (e.g. generate a secure token and migrate up)? If yes, that path reuses the token-security design. Owner: Claire.
- **How is AI cost gated for anonymous local users** so handing the app out doesn't expose an unbounded Anthropic bill? Owner: Claire + audit follow-up (Edge Function rate limiting).
- **Default on first open:** local by default, with cloud as an explicit choice? Owner: Claire.
- **The current no-token screen** becomes a "use locally" entry point instead of a blocker — confirm the desired first-run experience. Owner: Claire.
