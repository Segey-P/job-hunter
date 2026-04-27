# 05 — Roadmap

Phased build. Each phase is independently shippable (you can stop after any phase and have a useful tool). Each phase has explicit **learning goals** because that's the primary purpose of this project.

## Estimating philosophy

Estimates assume weekend/evening work with Claude Code as the coding partner, you reviewing and steering. They're learning-time, not pro-developer-time.

## Phase 1 — Skeleton extension (~1 weekend)

**Ships:** an extension that loads in Chrome, has an options page, has a popup with a "Fill" button that fills hardcoded test values into form fields by `autocomplete` attribute matching only.

**Includes:**
- Vite + `@crxjs/vite-plugin` build setup
- `manifest.json` (MV3, `activeTab` + `storage` permissions only)
- Service worker with message handler stub
- Content script injected on demand
- Options page (plain HTML form, no styling)
- Popup with "Fill" button

**Does not include:** encryption, ATS adapters, LLM, tracker.

**Learning goals:**
- MV3 lifecycle (worker sleep/wake)
- Manifest permissions model
- Message passing between content script ↔ worker
- `chrome.storage.local` reads/writes
- Build pipeline for an extension

**Done when:** loaded as unpacked extension, options page persists a dummy profile, popup's Fill button populates `<input autocomplete="email">` on any page.

## Phase 2 — Encrypted profile + first ATS adapter (~2 weekends)

**Ships:** profile is encrypted at rest, decrypted on passphrase entry. Greenhouse adapter fills real applications correctly.

**Includes:**
- Web Crypto API: PBKDF2 + AES-GCM (per `04-llm-integration.md`)
- Passphrase prompt in popup; session key cached in `chrome.storage.session`
- Auto-lock timer
- `AtsAdapter` interface and the dispatch layer
- Greenhouse adapter (start with `boards.greenhouse.io/*`)
- Visual highlight (green/yellow/red) post-fill

**Learning goals:**
- Web Crypto API: KDF and AEAD primitives, what `iv` and `tag` actually are
- Why React inputs need the native setter trick
- DOM stability of one specific ATS — what's safe to depend on

**Done when:** you can apply to a real Greenhouse posting end-to-end with the extension filling everything except the resume upload, screening questions, and submit.

## Phase 3 — Lever + heuristics + tracker (~2 weekends)

**Ships:** second ATS, generic heuristic fallback, application history.

**Includes:**
- Lever adapter
- Tier 2 heuristic dictionary and matcher
- Tracker: log every fill operation with `{company, role, url, ts, atsType, fieldsFilled, fieldsSkipped}`
- Popup tab to view/export tracker as JSON
- "Where did this come from?" — clicking a filled field shows which tier mapped it (debug overlay)

**Learning goals:**
- Generalizing across two adapters — what's actually common, what to extract
- Fuzzy matching tradeoffs (regex vs token similarity vs nothing)
- Designing a small log/tracker UI

**Done when:** Lever applications work, unknown ATSes get partial fill via heuristics, tracker shows a useful history.

## Phase 4 — LLM fallback (~1 weekend)

**Ships:** Tier 3 LLM call for unmapped fields.

**Includes:**
- `LlmProvider` interface
- One concrete provider (Gemini 2.5 Flash or Claude Haiku — pick during Phase 3 review)
- Encrypted API key storage
- Daily cap enforcement
- Debug log of LLM calls
- Settings UI for provider + key + cap

**Learning goals:**
- LLM API integration end-to-end (auth, JSON mode, error handling, rate limits)
- Prompt design for a constrained classification task
- Where the LLM helps vs. where it adds noise (you'll see the data)

**Done when:** an unfamiliar form (find one by hand — a small company's careers page) fills mostly correctly via LLM mapping, with low-confidence fields properly refused.

## Phase 5 — Workday adapter (~3 weekends, separate milestone)

**Ships:** Workday support.

Workday is its own beast. Multi-page, heavy SPA, slow, inconsistent across tenants. Treat as a learning exercise in itself.

**Approach:**
- Page detection (Workday URL patterns + DOM signature)
- Per-page state machine (My Information, My Experience, Voluntary Disclosures, Self Identify, Review)
- Wait for page-load idleness before filling each step
- Resume parsing handoff: Workday usually offers "Apply with resume" auto-parse — let it run, then fill/correct fields it got wrong rather than starting from blank

**Learning goals:**
- Adapting to a hostile (or at least indifferent) target site
- Patience with SPAs that fight you
- Knowing when to stop polishing and accept "80% works, 20% manual"

## Phase 6 — Optional: tailoring (open-ended)

Resume tailoring and cover letter generation. Different problem, different prompt, may not be worth doing — at 2-3 apps/week, manual tailoring with Claude in a separate window may be faster than building this in.

**Decide at the end of Phase 5 whether to start.**

## Stopping points

You can stop after any phase:

| Stop after | What you have |
|---|---|
| Phase 1 | Toy extension, learned MV3 |
| Phase 2 | Working tool for Greenhouse jobs only — and Greenhouse is common |
| Phase 3 | Useful for ~50% of postings (Greenhouse + Lever + decent heuristic fallback) |
| Phase 4 | Useful for ~70% — covers most non-Workday postings |
| Phase 5 | Covers ~85% of postings you'd realistically apply to |

Phase 3 is probably the natural finish line at your application volume. Phase 5 is for the learning, not the time savings.
