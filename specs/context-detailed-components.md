# 01 — Architecture

## Components

```
┌────────────────────────────────────────────────────────────────┐
│  Chrome Browser                                                │
│                                                                │
│  ┌──────────────┐    ┌──────────────────────────────────────┐  │
│  │ Job posting  │◄───┤ Content Script (injected per page)   │  │
│  │ DOM          │    │  - Form detection                    │  │
│  └──────────────┘    │  - ATS adapter dispatch              │  │
│                      │  - Generic heuristic fill            │  │
│                      │  - Apply mappings to inputs          │  │
│                      └────────────┬─────────────────────────┘  │
│                                   │ chrome.runtime messages    │
│  ┌────────────────────────────────▼─────────────────────────┐  │
│  │ Service Worker (background, MV3)                         │  │
│  │  - Profile decrypt/encrypt                               │  │
│  │  - LLM API call (fallback only)                          │  │
│  │  - Application tracker writes                            │  │
│  └────────────────────────────────┬─────────────────────────┘  │
│                                   │                            │
│  ┌────────────────────────────────▼─────────────────────────┐  │
│  │ chrome.storage.local                                     │  │
│  │  - profile.enc (AES-GCM ciphertext)                      │  │
│  │  - applications.json                                     │  │
│  │  - settings.json (LLM provider, API key encrypted)       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │ Popup UI     │  │ Options UI   │                            │
│  │ (Fill button,│  │ (Profile     │                            │
│  │  tracker)    │  │  editor,     │                            │
│  └──────────────┘  │  passphrase) │                            │
│                    └──────────────┘                            │
└────────────────────────────────────────────────────────────────┘
                                   │
                                   │ HTTPS (only field metadata)
                                   ▼
                        ┌──────────────────────┐
                        │ LLM API (Anthropic   │
                        │ or Gemini)           │
                        │ — fallback only      │
                        └──────────────────────┘
```

## Component responsibilities

| Component | Responsibility | Privacy concern |
|---|---|---|
| Content script | Read DOM, write to inputs, dispatch events | Has access to profile values in memory while filling |
| Service worker | Decrypt profile, call LLM, persist tracker | Holds passphrase-derived key in memory only |
| Options page | Edit profile, set passphrase, configure LLM | User-facing form, never sends data anywhere |
| Popup | Trigger fill, view recent applications | Read-only window into worker state |
| `chrome.storage.local` | At-rest storage | Always encrypted for profile and API key |

## Why MV3 service worker (not background page)

MV2 is deprecated. MV3 service workers are ephemeral — they sleep when idle and wake on events. This means:

- The decryption key cannot be held forever; we re-prompt for passphrase per session, OR cache the derived key with a short TTL in `chrome.storage.session` (which is in-memory, cleared on browser close).
- LLM API calls and tracker writes are event-driven, which fits the worker lifecycle naturally.

## Why content script per page (not a single global script)

Each tab gets its own content-script context. We only inject on user trigger (clicking the popup's Fill button), not on every page load — this avoids:

- Reading every page's DOM unnecessarily
- Scope creep into "auto-detect job pages and pop up"
- Permissions surface (manifest declares `activeTab`, not broad host permissions)

## Message passing

Content script ↔ service worker uses `chrome.runtime.sendMessage` with a small typed protocol:

```
{ type: 'GET_PROFILE_FOR_FILL', site: 'greenhouse' }
{ type: 'CLASSIFY_FIELDS', fields: [{ label, type, options }] }
{ type: 'LOG_APPLICATION', payload: { company, role, url, ts } }
```

All messages are validated against a schema in the worker. Unknown messages are dropped.

## Out of scope for v1

- Cross-device sync
- Cloud backup
- Multiple profiles (single user, single profile)
- Browser support beyond Chrome (Firefox MV3 is close-enough to port later if wanted)
- Mobile
