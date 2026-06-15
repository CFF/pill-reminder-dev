# Posology — Design System

The visual system: identity, color, type, component conventions, and known design debt. For build details see **SPECS.md**. For behavioral rules see **CLAUDE.md**.

## Identity

iOS-native feel, warm and calm. This is a daily health tool used by patients, often anxiously, sometimes one-handed. The design serves the task: read fast, tap confidently, never feel clinical or cold.

Core signals:
- **Orange (`--acc`) = in progress.** Green (`--green`) = fully complete. This pairing carries the app's primary status language.
- Warm paper background, grouped list cards, inset separators, generous tap targets.
- Warmth comes from the accent and the serif display type, not from piling on more cream surfaces.

## Color

Auto light/dark via `prefers-color-scheme`. Real values below.

### Light

| Token | Value | Role |
|---|---|---|
| `--bg` | `#f7ede0` | App background (warm paper) |
| `--bg2` | `#e8ddd1` | Recessed surface, assistant bubbles |
| `--card` | `#ffffff` | Cards, headers, input bars |
| `--card2` | `#faf7f3` | Secondary card surface |
| `--t1` | `#18181b` | Primary text |
| `--t2` | `#5c5c66` | Secondary text |
| `--t3` | `rgba(0,0,0,0.28)` | Tertiary. **Decorative only, never on text that must be read** |
| `--acc` | `#d45500` | Accent, in-progress, primary actions |
| `--acc-soft` | `rgba(212,85,0,0.10)` | Soft accent fill |
| `--acc-glow` | `rgba(212,85,0,0.22)` | Accent glow |
| `--green` | `#1eb854` | Complete |
| `--blue` | `#0066e8` | New / informational |
| `--red` | `#e5342c` | Destructive |
| `--sep` / `--sep-strong` | `rgba(0,0,0,0.065)` / `rgba(0,0,0,0.13)` | Separators (use strong for disclaimers, tip card borders) |

### Dark

| Token | Value |
|---|---|
| `--bg` / `--bg2` | `#161618` / `#232325` |
| `--card` / `--card2` | `#252528` / `#2e2e32` |
| `--t1` / `--t2` / `--t3` | `#f0f0f5` / `#a0a0ab` / `rgba(255,255,255,0.26)` |
| `--acc` / `--acc-soft` / `--acc-glow` | `#ff6b1a` / `rgba(255,107,26,0.14)` / `rgba(255,107,26,0.28)` |
| `--blue` / `--green` / `--red` | `#3b9eff` / `#2ed15f` / `#ff4d45` |
| `--sep` / `--sep-strong` | `rgba(255,255,255,0.08)` / `rgba(255,255,255,0.15)` |

## Typography

- **Titles:** DM Serif Display (`.serif`, falls back to Georgia). The warm, human voice of the app.
- **Body:** DM Sans (falls back to system-ui). Not a system font, not Inter, despite what older docs said.
- Loaded from Google Fonts.

## Component conventions

- Pill cards: border-radius 18px. Group cards: 16px.
- Tap targets: 44px minimum (iOS HIG).
- `env(safe-area-inset-*)` on all edges.
- Primary action: full-width accent button. Cancel: full-width text link below, never side by side.
- Subtle card shadow + light border stroke.

## Known design debt

Captured from an impeccable critique. Not yet fixed.

1. **`--t3` on small text fails contrast.** Tertiary text on tinted near-white lands around 1.9:1, well under the 4.5:1 needed for body. The rule: never put readable text in `--t3`. Use `--t2` as the floor. Audit any small label, disclaimer, or caption currently in `--t3`.
2. **The warm-paper background sits in the band that reads as "AI default" in 2026.** It is a committed brand identity, so it stays. The discipline: do not reflexively add new cream or ivory tinted surfaces. Carry warmth through accent and type, not more beige.
3. **Nurse tab uses the most generic chatbot archetype** (robot avatar, three bouncing dots, suggestion chips). Tonal mismatch with the "caring nurse" promise. Queued to warm up.
4. **Nurse tab repeats its disclaimer three times** (subtitle, banner, per-message footer). Keep one strong placement.
5. **Nurse suggestion "Can I stop early?"** steers users toward a question the assistant is forbidden to answer. Replace with safer prompts.
