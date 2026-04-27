# job-hunter

A Manifest V3 Chrome extension to intelligently fill job application forms while maintaining user privacy and control.

**Status:** Phase 1 (Design)  
**Owner:** Segey-P  
**Repo:** github.com/Segey-P/job-hunter

## Overview

Saves time on repetitive job applications by auto-filling forms using a local encrypted profile. The extension uses a three-tier strategy:
1. **Hardcoded ATS adapters** (Greenhouse, Lever, Ashby, etc.) — deterministic, no LLM
2. **Generic heuristics** — autocomplete, type, name/id fuzzy matching
3. **LLM fallback** (Gemini/Claude) — semantic mapping for unmapped fields

All data stays local. Only field metadata sent to LLM. **No auto-submit—always manual review before submission.**

## Key Features

- Local-first encrypted profile (AES-GCM with passphrase)
- Three-tier field mapping strategy (adapters → heuristics → LLM)
- Application tracker (history of what you applied to)
- Multi-ATS support (Greenhouse, Lever, Ashby, iCIMS, Workday)
- Zero cost (uses Gemini free tier or Claude Haiku)
- Privacy-first (PII never leaves your machine)

## Quick Start

**Prerequisites:** Node.js 18+, Chrome browser

1. Clone: `git clone https://github.com/Segey-P/job-hunter.git && cd job-hunter`
2. Install: `npm install` (when Phase 1 is done)
3. Build: `npm run build` → outputs to `dist/`
4. Load: Visit `chrome://extensions`, toggle "Developer mode", click "Load unpacked", select `dist/`

## Project Structure

```
specs/
  ├── spec-data-model.md           ← Profile + tracker JSON schemas
  ├── spec-fill-strategy.md        ← Three-tier filling logic
  ├── spec-llm-integration.md      ← LLM provider, encryption, privacy
  ├── plan-phases.md               ← Phased development roadmap (6 phases)
  ├── ref-decisions.md             ← Architecture decisions (ADR log)
  └── context-detailed-components.md ← Component responsibilities

README.md                           ← This file
CLAUDE.md / AGENTS.md               ← Agent instructions
TODO.md                             ← Active work items
```

## Development

**Always read the specs before starting work:**
- `specs/spec-data-model.md` — understand the profile schema
- `specs/spec-fill-strategy.md` — understand the three-tier approach
- `specs/plan-phases.md` — understand the roadmap
- `specs/ref-decisions.md` — understand past decisions

See `TODO.md` for current phase and next actions.
