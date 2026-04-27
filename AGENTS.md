# Agent Context — job-hunter

**Agent-neutral mirror of CLAUDE.md. Use this when other agents (Gemini, Cursor, Aider) need context.**

## What This Project Is

A Manifest V3 Chrome extension to save time on job applications by intelligently filling forms. Uses a local encrypted profile + three-tier fill strategy (ATS adapters → heuristics → LLM fallback). **No auto-submit; always manual review.**

## Project Overview

- **Name:** job-hunter
- **Status:** Phase 1 (Design)
- **Repo:** github.com/Segey-P/job-hunter
- **Stack:** TypeScript + Vite, Chrome MV3, Service Worker, Content Scripts, Web Crypto
- **LLM:** Gemini 2.5 Flash (free) or Claude Haiku (privacy preferred)
- **Deploy:** Local unpacked extension (not Chrome Web Store)

## User Profile

- **Role:** Senior IT Consultant (Banking), Canadian (Squamish, BC — Pacific Time)
- **Style:** Direct, precise. Tables and bullet points. No softening.
- **Technical level:** Not a developer — plain language for options.

## Core Design Decisions

1. **Chrome extension, not separate agent.** Fits "browse → see posting → fill it" workflow.
2. **Three-tier fill strategy.** Adapters (deterministic) → heuristics (free) → LLM (only unmapped). Most forms never call the LLM.
3. **No auto-submit.** User manually clicks submit. Catches mistakes in screening questions.
4. **JSON profile, not Markdown.** Structured data: years per skill, work auth per region, etc.
5. **Encrypted at rest.** AES-GCM, key derived from passphrase (PBKDF2 600k iter), cached in `chrome.storage.session` only.
6. **Self-hosted.** User controls every line handling their PII.

## Rules (All Agents)

1. Read `TODO.md` first — source of truth for next actions.
2. Read `specs/` in this order before making changes:
   - `spec-data-model.md` — profile/tracker schemas
   - `spec-fill-strategy.md` — three-tier logic
   - `spec-llm-integration.md` — LLM provider, encryption
   - `plan-phases.md` — roadmap
   - `ref-decisions.md` — decision log
3. Never commit secrets (API keys, passphrases).
4. Push to `main` directly.
5. Update `TODO.md` throughout the session (not at the end).
6. Prefer editing over creating files.
7. Keep dependencies minimal.
8. Update specs if requirements/behavior change materially.

## Key Files

| File | Purpose |
|---|---|
| `TODO.md` | Next actions (read by Project Hub) |
| `specs/spec-data-model.md` | Profile/tracker JSON schemas |
| `specs/spec-fill-strategy.md` | Three-tier fill logic, adapters, heuristics |
| `specs/spec-llm-integration.md` | LLM provider, encryption, privacy |
| `specs/plan-phases.md` | Phase 1–6 roadmap |
| `specs/ref-decisions.md` | ADR log |
| `specs/context-detailed-components.md` | Component responsibilities |

## Tech Notes

### MV3 & Service Worker
- Ephemeral worker (sleeps when idle, wakes on events).
- Passphrase key cached in `chrome.storage.session` (in-memory, cleared on browser close).
- Auto-lock after 30 min.

### Content Script
- Injected only on Fill button click (not on every page load).
- Uses `activeTab` permission (minimal surface).
- Communicates with service worker via `chrome.runtime.sendMessage`.

### ATS Adapters
- One adapter per ATS (Greenhouse, Lever, Ashby, iCIMS, Workday).
- Contract: `matches(url, dom)`, `detectFields(dom)` → `DetectedField[]`.
- No fill execution in adapters — just detection.

### Profile Schema
- `identity`, `address`, `links`, `workAuth` (per region), `compensation`, `demographics`, `experience`, `education`, `skills`, `standardAnswers`.
- See `specs/spec-data-model.md` for full schema.

### Privacy Guarantees
- Profile values never sent to LLM.
- Only field metadata and available profile keys.
- Form URL, company name, role never sent.
- API key never accessible to content scripts.

## Phases

| Phase | Ships | Est. | Stoppoint? |
|-------|-------|------|-----------|
| 1 | MV3 skeleton, options page, dummy fills | 1 wk | Toy |
| 2 | Encrypted profile, Greenhouse adapter | 2 wk | Greenhouse only |
| 3 | Lever + heuristics + tracker | 2 wk | ~50% postings |
| 4 | LLM fallback | 1 wk | ~70% postings |
| 5 | Workday adapter | 3 wk | ~85% postings |

**Current:** Phase 1 (code to start).
