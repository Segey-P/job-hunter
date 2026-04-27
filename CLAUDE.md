# Claude Context — job-hunter

## Project Overview

- **Name:** job-hunter
- **Status:** Phase 1 (Design)
- **Repo:** github.com/Segey-P/job-hunter
- **Purpose:** Manifest V3 Chrome extension to auto-fill job applications while maintaining privacy and user control.
- **Stack:** TypeScript + Vite, Chrome MV3, Service Worker, Content Scripts, Web Crypto API
- **Deploy:** Local unpacked extension (not Chrome Web Store)
- **LLM:** Gemini 2.5 Flash (free) or Claude Haiku (if privacy posture demands)

## User Profile

- **Role:** Senior IT Consultant (Banking sector), Canadian (Squamish, BC — Pacific Time)
- **Style:** Direct, precise, no softening. Tables and bullet points.
- **Technical level:** Not a developer — use plain language when explaining options.
- **Workflow:** Cloud-first via Claude Code web. Minimize local machine dependency.

## Core Design Decisions

1. **Chrome extension, not separate agent.** Fits the "I browse, I see a posting, I fill it" workflow. No separate browser. Keeps existing logins/cookies.
2. **Three-tier fill strategy.** ATS adapters (deterministic) → heuristics (no LLM cost) → LLM fallback (only unmapped fields). Most forms never trigger an LLM call.
3. **No auto-submit, ever.** User manually clicks the submit button. Screening question mistakes and edge cases get caught by human review.
4. **Profile is JSON, not Markdown.** Structured data (years per skill, work auth per region). Markdown can be a future import format.
5. **Encrypted at rest.** AES-GCM with passphrase-derived key (PBKDF2 600k iterations). Session key cached in `chrome.storage.session` only.
6. **Self-hosted, not Web Store.** User controls every line of code that handles their PII.

See `specs/ref-decisions.md` for the full ADR log.

## Before Making Changes

1. Read `TODO.md` — current phase and next actions.
2. Read these specs in order:
   - `specs/spec-data-model.md` — profile/tracker JSON schemas
   - `specs/spec-fill-strategy.md` — three-tier logic, ATS adapters, heuristics
   - `specs/spec-llm-integration.md` — LLM provider, encryption, privacy guarantees
   - `specs/plan-phases.md` — phased roadmap (Phase 1–6)
   - `specs/ref-decisions.md` — decisions made and why
3. Never commit secrets (API keys, passphrases, private config).

## AI Agent Rules

1. Push directly to `main` — no feature branches unless large/risky.
2. Prefer editing existing files over creating new ones.
3. No unnecessary dependencies — keep the stack minimal and modern.
4. No comments explaining *what* code does — only *why*, when non-obvious.
5. No premature abstractions. Three similar lines beats a helper function.
6. No half-finished implementations. Either complete it or don't start.
7. Update `TODO.md` throughout the session, not at the end.
8. Update specs in `specs/` if requirements or behavior change materially.

## TODO.md Convention

`TODO.md` is the source of truth. The Project Hub dashboard reads it and shows items to the user.

Keep it:
- **Short.** 5–6 items max. This is "next" work, not a backlog.
- **Active.** Delete completed items immediately; don't accumulate.
- **Current.** Update as you work, not just at the end.

Format:
```
- [ ] Action item (present tense)
- [ ] Another action item
```

## Key Files

| File | Purpose |
|---|---|
| `TODO.md` | Next actions (read by Project Hub) |
| `CLAUDE.md` / `AGENTS.md` | Agent instructions |
| `specs/spec-data-model.md` | Profile/tracker JSON schemas |
| `specs/spec-fill-strategy.md` | Three-tier fill logic, ATS adapters, heuristics |
| `specs/spec-llm-integration.md` | LLM provider, encryption, key derivation |
| `specs/plan-phases.md` | Phase 1–6 roadmap (stoppoints at 1, 2, 3, 4, 5) |
| `specs/ref-decisions.md` | ADR log — decisions and rejections |
| `specs/context-detailed-components.md` | Component responsibilities, storage layout |

## Technology Notes

### MV3 & Service Worker
- Service worker is ephemeral — sleeps when idle, wakes on events.
- Passphrase-derived key cannot be held across sleep cycles.
- Solution: Cache key in `chrome.storage.session` (in-memory, cleared on browser close).
- Auto-lock after 30 min idle.

### Content Script Injection
- Injected only on user trigger (Fill button click), not on every page load.
- Uses `activeTab` permission (minimal surface).
- Dispatches messages to service worker for decryption, LLM calls.

### ATS Adapters
- One adapter per ATS (Greenhouse, Lever, Ashby, iCIMS, Workday).
- Adapter contract: `matches(url, dom)`, `detectFields(dom)` → `DetectedField[]`.
- No fill execution in adapters — just detection.

### Profile Schema (v1)
- `identity` — name, email, phone
- `address` — street, city, region, postal code (defaults to Squamish, BC)
- `links` — LinkedIn, GitHub, portfolio
- `workAuth` — per-region (CA, US, UK, EU) + sponsorship flag
- `compensation` — salary range (CAD), notice period, relocation/travel
- `demographics` — gender, ethnicity, veteran, disability (all default to "decline")
- `experience` — array of roles with company, title, dates, summary
- `education` — array of institutions, degrees, fields
- `skills` — technical, domain, languages (with years/proficiency)
- `standardAnswers` — pre-written responses to common screening questions

### Privacy Guarantees
- Profile values never sent to LLM — only field metadata and available keys.
- Form URL, company name, role title not sent.
- API key never accessible to content scripts.
- Training data: verify Gemini free-tier terms before Phase 4. If uneasy about Google training on free data, use Claude Haiku instead.

## Development Phases (see `specs/plan-phases.md`)

| Phase | Ships | Estimate | Stoppoint? |
|-------|-------|----------|-----------|
| 1 | MV3 skeleton, options page, dummy profile, hardcoded test fills | 1 weekend | Toy extension |
| 2 | Encrypted profile, passphrase, Greenhouse adapter | 2 weekends | Working tool for Greenhouse only |
| 3 | Lever + heuristics + tracker | 2 weekends | ~50% of postings (Greenhouse/Lever) |
| 4 | LLM fallback (Gemini/Claude) | 1 weekend | ~70% of postings |
| 5 | Workday adapter | 3 weekends | ~85% of postings |
| 6 | Cover letter/resume tailoring (optional) | Open-ended | Not committing to this |

Current phase: **Phase 1** (design complete, code to start)
