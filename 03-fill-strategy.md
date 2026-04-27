# 03 — Fill Strategy

Three tiers, attempted in order. Each field is owned by exactly one tier — the first that claims it wins.

## Tier 1: ATS adapters (deterministic)

For known ATSes, hardcoded selectors map known fields to profile keys. Zero LLM calls. Fast and reliable.

### Adapter contract

```ts
interface AtsAdapter {
  name: string                      // 'greenhouse' | 'lever' | etc.
  matches(url: string, dom: Document): boolean
  detectFields(dom: Document): DetectedField[]
}

interface DetectedField {
  element: HTMLElement              // input, select, textarea
  profileKey: string | null         // e.g. 'identity.email'; null = unknown
  confidence: 'high' | 'medium'     // adapters never produce 'low'
  type: 'text' | 'select' | 'radio' | 'checkbox' | 'textarea'
  options?: string[]                // for selects/radios
}
```

### Adapter rollout order

1. **Greenhouse** — most consistent DOM, easiest to start. `boards.greenhouse.io/*` and embedded forms.
2. **Lever** — also clean. `jobs.lever.co/*`.
3. **Ashby** — newer, widely used in tech.
4. **iCIMS** — common in banking and enterprise.
5. **Workday** — last. Multi-page wizard, heavy SPA, the hardest. Treat as its own milestone.

### What goes in an adapter

Only the field-mapping logic. No fill execution (that's shared). No DOM mutation outside reading. Adapters are pure data extractors.

## Tier 2: Generic heuristics (no LLM)

For unknown sites or fields the adapter missed. Zero cost, zero network.

Match strategy in priority order:

1. **`autocomplete` attribute** — `email`, `tel`, `given-name`, `family-name`, `street-address`, `postal-code`, etc. These are spec'd HTML and reliable when present.
2. **`type` attribute** for inputs — `type="email"` → `identity.email`, `type="tel"` → `identity.phone`.
3. **`name` and `id` attribute** — fuzzy match against a keyword dictionary (e.g. `/email/i` → `identity.email`, `/first.?name/i` → `identity.firstName`).
4. **Associated `<label>` text** — same fuzzy dictionary applied to the label content. Resolves via `for=` or DOM proximity.
5. **`placeholder`** — last resort, often unreliable.

### Keyword dictionary (initial)

| Pattern | Profile key |
|---|---|
| `/^email$|email.address/i` | `identity.email` |
| `/first.?name|given.?name/i` | `identity.firstName` |
| `/last.?name|family.?name|surname/i` | `identity.lastName` |
| `/^phone$|mobile|telephone/i` | `identity.phone` |
| `/linked.?in/i` | `links.linkedin` |
| `/git.?hub/i` | `links.github` |
| `/portfolio|website/i` | `links.portfolio` |
| `/city|town/i` | `address.city` |
| `/postal|zip/i` | `address.postalCode` |
| `/country/i` | `address.country` |
| `/notice.?period/i` | `compensation.noticePeriodWeeks` |
| `/sponsor/i` | `workAuth.requiresSponsorship` |

This list grows as you encounter forms. Versioned in code, not config.

## Tier 3: LLM fallback (semantic mapping)

For fields neither tier 1 nor tier 2 could map. Send field metadata, get back a mapping. Profile values never leave the machine.

Details in `04-llm-integration.md`. Summary:

- Batched: one LLM call per form, not per field.
- Input: array of `{label, type, options, name, id}` for unmapped fields only.
- Output: array of `{label, profileKey | null, confidence, suggestedValue?}`.
- Cached: `(label_hash) → profileKey` cached for the session so repeat fields skip the call.

## Fill execution (shared across tiers)

After mapping, fill is uniform regardless of tier:

```ts
function fillField(el: HTMLElement, value: string, type: FieldType) {
  // For React/Vue inputs: setting .value alone doesn't trigger re-render
  // Use the native setter then dispatch input + change events
  const setter = Object.getOwnPropertyDescriptor(
    el.constructor.prototype, 'value'
  )?.set
  setter?.call(el, value)
  el.dispatchEvent(new Event('input', { bubbles: true }))
  el.dispatchEvent(new Event('change', { bubbles: true }))
}
```

This pattern is necessary for any modern SPA. The naive `el.value = x` works on plain HTML but silently fails on React-controlled inputs — the framework re-renders and overwrites.

### Visual confirmation (HITL)

After fill, briefly highlight each filled field with an outline (1.5s fade) so the user can see what was touched. Skipped fields get a different highlight color. This is the actual human-in-the-loop UX — not "press a button to confirm submit," but "see at a glance what got filled and what didn't."

## Confidence and refusal

The fill engine has three confidence levels per field:

| Confidence | Behavior |
|---|---|
| `high` | Fill silently, highlight green |
| `medium` | Fill, highlight yellow, log to popup |
| `low` (LLM uncertain) | **Do not fill.** Highlight red, log "needs review" |

We don't fill low-confidence guesses. False fills are worse than no fill — they hide problems the user would otherwise catch.

## What this strategy does not handle

- **File uploads** — out of scope. Manual.
- **CAPTCHAs** — out of scope. Will halt fill, alert user.
- **Multi-page wizards** — partial. Tier 1 adapters can handle Workday-style multi-page if specifically coded. Tier 2/3 fill one page at a time; user clicks Next, re-triggers fill.
- **Custom open-text questions** ("Why do you want this role?") — out of scope for v1. Possibly addressed by Phase 5 tailoring feature.
- **Skill matrices / "rate yourself 1-5 on each"** — out of scope. Manual.
