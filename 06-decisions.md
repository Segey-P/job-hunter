# 06 — Decisions

ADR-style log. Each decision records what was chosen, what was rejected, and why. Update when decisions change.

---

## D1 — Chrome extension over Playwright agent

**Status:** Accepted
**Date:** 2026-04

**Context:** The original Gemini-generated spec proposed a Python orchestrator using `browser-use` + Playwright + stealth plugins driving a separate Chromium instance.

**Decision:** Build a Manifest V3 Chrome extension that runs inside the user's daily Chrome.

**Rationale:**
- The "I browse, I see a posting, I fill it" workflow doesn't fit a separate-browser agent.
- Playwright would lose the user's existing logins, cookies, LinkedIn session.
- Stealth plugins solve a problem (bot detection at scale) the user doesn't have at 2-3 apps/week.
- Lower complexity, fewer moving parts, smaller attack surface.

**Rejected:**
- **Playwright + browser-use** — wrong shape; built for unattended automation.
- **Tampermonkey userscript** — viable but caps long-term capability (no service worker, no encrypted storage, weaker manifest).
- **Native desktop app with embedded browser** — overkill.

---

## D2 — LLM as Tier 3 fallback only, not primary mechanism

**Status:** Accepted
**Date:** 2026-04

**Context:** Gemini's spec used `browser-use`, which queries an LLM at every interaction step.

**Decision:** Three-tier fill strategy. Hardcoded ATS adapters first. Generic heuristics second. LLM only for fields neither tier mapped, and only one batched call per form.

**Rationale:**
- Top ATSes have stable DOM. Hardcoded selectors are *more* reliable than LLM guessing, not less.
- Per-field LLM calls would burn free-tier limits within a few applications.
- Determinism matters when humans review fills — surprising LLM outputs erode trust faster than missing fills.

**Rejected:**
- **LLM-first vision agent** — slow, expensive, brittle on the hardest sites.
- **Pure heuristic / no LLM at all** — leaves too many real fields unmapped on long-tail sites.

---

## D3 — JSON as profile source of truth, not Markdown

**Status:** Accepted
**Date:** 2026-04

**Context:** Gemini's spec proposed `profile.md` as the master profile.

**Decision:** Profile is a versioned JSON object. Markdown can be a future import format if useful. The options page UI is the primary editor.

**Rationale:**
- Programmatic fields (years per skill, work auth per region, demographic answers) need structure.
- Free-form Markdown invites parsing brittleness — exactly the kind of "you'll spend an afternoon debugging the parser" that doesn't serve learning goals.
- Schema versioning is impossible without a schema.

**Rejected:**
- **Markdown profile** — fine for resume content, bad as machine-readable identity.
- **YAML** — JSON is fine, no need for YAML's ambiguity.

---

## D4 — No auto-submit, ever

**Status:** Accepted
**Date:** 2026-04

**Context:** Common feature in autonomous-application tools.

**Decision:** Extension fills only. Submit is always manual.

**Rationale:**
- Screening questions and edge cases will sometimes get filled wrong; user review catches it.
- Submitting bad data costs you the role. There is no "undo a submitted application."
- Personal use, low volume — saving the 5 seconds of clicking Submit is not worth the risk.

**Not revisitable.**

---

## D5 — Encryption at rest using Web Crypto + passphrase

**Status:** Accepted
**Date:** 2026-04

**Context:** Profile contains PII that shouldn't be casually readable from `chrome.storage.local`.

**Decision:** AES-GCM, key derived from user passphrase via PBKDF2-SHA-256 with 600k iterations. Session key cached in `chrome.storage.session` only.

**Rationale:**
- Chrome doesn't encrypt extension storage on disk by default.
- Web Crypto is built-in, audited, no extra dependency.
- 600k iterations is the OWASP 2023 recommendation for PBKDF2-SHA-256.

**Rejected:**
- **No encryption** — fails the user's "no PII on github obviously" privacy posture, even though storage isn't on github. Same posture should apply to disk.
- **Argon2id** — better algorithm but not in Web Crypto; would need a WASM library, more dependency for marginal gain.
- **Sync to `chrome.storage.sync`** — couples to Google account sync; explicitly avoided.

---

## D6 — Build from source, do not publish

**Status:** Accepted
**Date:** 2026-04

**Context:** Personal-use tool with PII handling.

**Decision:** Load as unpacked extension from local build. Don't publish to Chrome Web Store.

**Rationale:**
- Chrome Web Store publication adds review overhead and zero value for a single user.
- Self-built means the user sees and controls every line of code that handles their data.
- Sidesteps the Web Store's evolving (and tightening) policies on auto-fill / job-application extensions.

---

## D7 — One LLM provider abstraction, start with Gemini 2.5 Flash, swap to Claude Haiku if privacy posture demands

**Status:** Accepted (provisional)
**Date:** 2026-04

**Context:** Free tier vs paid, training-on-data concerns.

**Decision:** `LlmProvider` interface with one concrete impl at a time. Start Gemini for cost. Verify current Google free-tier training/retention policy before committing to it; if uncomfortable, switch to Claude Haiku (paid, but pennies/month at this volume).

**Action item before Phase 4:** read current Gemini API free-tier terms.

---

## D8 — No file uploads, no captchas, no multi-page wizard support in v1

**Status:** Accepted
**Date:** 2026-04

**Context:** Scope discipline.

**Decision:** v1 fills single-page forms. Resume upload, captchas, multi-page Workday flows are explicitly out for Phase 1–4. Workday is its own Phase 5.

**Rationale:** Building these badly is worse than not building them. Scope creep killed many side projects.

---

## Open questions

- Q1: Which OS is the user on for development? (Affects Node setup instructions in Phase 1.)
- Q2: Cap on `experience.roles` array length in the schema? (Probably not needed but worth noting.)
- Q3: Should the tracker support manual edits (post-hoc notes) or be read-only?
