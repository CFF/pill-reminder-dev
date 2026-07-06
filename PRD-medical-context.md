# Posology — PRD: Medical Context / Nurse Memory

## Objective

Give the Nurse enough context about the user that its answers are relevant from the very first question, without the user having to re-explain themselves in every conversation.

This is not a medical record. It is a light, stable memory that the user controls.

---

## Scope

Three complementary layers, all already partially in place or decided:

| Layer | Status |
|---|---|
| `reason` field on each tracker medication | In dev (`feat/pill-reason`), merged to main dev |
| "Medical context" section in Settings | To implement |
| Collapsed context banner at the top of the Nurse tab | To implement |
| Injection into the `nurse-chat` system prompt | To implement |

---

## 1. `reason` field on medications

Already documented in the tracker. Each tracker entry can carry a free-text `reason` field (e.g. "For migraines", "Post-op anticoagulant"). Shown as a neutral gray chip at the bottom of the medication card.

Injected into the system prompt as a list: `- Doliprane 1g (morning) — For chronic pain`.

---

## 2. "Medical context" section in Settings

A single editing surface for stable data. The user fills it in once, updates it if it changes.

### Fields

| Field | Type | Limit | Placeholder |
|---|---|---|---|
| Chronic conditions | Free text | 300 char. | "e.g. type 2 diabetes, hypothyroidism…" |
| Known allergies and intolerances | Free text | 200 char. | "e.g. penicillin, aspirin, lactose…" |
| Important history | Free text | 300 char. | "e.g. heart surgery 2019, pregnancy…" |
| Free notes | Free text | 500 char. | "Anything that can help the Nurse answer better" |

### Rules

- All fields are optional
- Auto-save (no separate "Save" button) or a single button at the bottom of the section
- Stored in Supabase under the key `pr:medical-context` (same user namespace)
- No field is displayed anywhere else in the app except in the Nurse banner

---

## 3. Context banner in the Nurse tab

Shown at the top of the Nurse tab, read-only. Collapsed by default.

### Behavior

- Default state: collapsed, one summary line visible ("Medical context · 3 items filled in")
- Expandable by tap to see the full detail
- If no field is filled in: banner absent (no empty placeholder)
- "Edit" link opens the corresponding Settings section directly
- Read-only — editing happens exclusively in Settings

### Content shown when expanded

- Chronic conditions (if filled in)
- Allergies (if filled in)
- History (if filled in)
- Free notes (if filled in)
- List of active medications with their `reason` (if filled in)

---

## 4. Enriched `nurse-chat` system prompt

The system prompt is built client-side and sent to `nurse-chat` via the existing `system` field. The Edge Function passes it directly to the Anthropic API without modification.

### Prompt structure

```
You are the Posology Nurse, a caring and informed health assistant.
You help users understand their medications, their possible interactions,
and the precautions to take — without ever replacing a doctor's or pharmacist's advice.

BEHAVIOR RULES:
- You do not prescribe, diagnose, or modify treatments.
- For any medical decision, you refer the user to their doctor or pharmacist.
- You stay factual, calm, and reassuring.
- If a question is beyond your role, you say so clearly and directly.

CONFIDENTIALITY:
- The data below is provided by the user to improve the relevance of your answers.
- You do not repeat this data back to the user unless it is useful to the answer.
- You do not share it with anyone and do not mention it unnecessarily.

USER PROFILE:
[If filled in] First name: {{profile.name}}
[If filled in] Age: {{profile.age}} years
[If filled in] Weight: {{profile.weight}} kg

MEDICAL CONTEXT:
[If filled in] Chronic conditions: {{medicalContext.conditions}}
[If filled in] Allergies and intolerances: {{medicalContext.allergies}}
[If filled in] Important history: {{medicalContext.history}}
[If filled in] Notes: {{medicalContext.notes}}

ACTIVE TREATMENT:
[Dynamically generated list]
- {{pill.name}} {{pill.dose}} ({{session}})[ — {{pill.reason}}]
```

### Injection rules

- `[If filled in]` blocks are omitted if the field is empty or null — never an empty line in the prompt
- A medication's `reason` field is omitted if absent — no orphan dash
- The profile is omitted entirely if all profile fields are empty
- The "Medical context" block is omitted entirely if all fields are empty
- Built client-side in the function that calls `nurse-chat`

---

## Storage

| Supabase key | Content |
|---|---|
| `pr:medical-context` | `{ conditions, allergies, history, notes }` — strings or null |
| `pr:profile` | Existing — `{ name, age, weight, height }` |
| `pr:list` | Existing — includes `reason` on each entry |

---

## Out of scope

- Validation or structuring of entries (no ICD condition dropdown, free text only)
- Sharing medical context between users
- Exporting the medical context
- History of context changes

---

## Dependencies

- `feat/pill-reason` merged to main dev: required (the `reason` field must exist on tracker entries)
- Existing `pr:profile`: used as is
- `nurse-chat` Edge Function: no modification required (already accepts `system` as a parameter)
