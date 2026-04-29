# job-hunter — TODO

## Phase 1 — MV3 Skeleton & Options Page

### Outstanding Work
- [ ] Implement MV3 service worker setup + message routing
- [ ] Build options page UI (Chrome options protocol)
- [ ] Implement passphrase entry & encryption (AES-GCM PBKDF2)
- [ ] Hardcoded profile + dummy Greenhouse test form
- [ ] Content script injection on /fill button click

### Before Starting
**On session start, read specs in this order:**
1. `specs/spec-data-model.md` — profile/tracker schemas
2. `specs/spec-fill-strategy.md` — three-tier fill logic  
3. `specs/spec-llm-integration.md` — LLM provider & encryption
4. `specs/plan-phases.md` — roadmap & phase breakdown
5. `specs/ref-decisions.md` — decisions & ADR log

## Future Phases

- **Phase 2:** Greenhouse adapter + encrypted profile storage (2 weekends)
- **Phase 3:** Lever + heuristics + application tracker (2 weekends)
- **Phase 4:** LLM fallback (Gemini/Claude) (1 weekend)
- **Phase 5:** Workday adapter (3 weekends)
- **Phase 6:** Cover letter/resume tailoring (optional)
