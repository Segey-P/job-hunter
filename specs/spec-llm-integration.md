# 04 — LLM Integration

LLM is **fallback only** — Tier 3 in the fill strategy. Most forms should never trigger a call.

## When the LLM is called

- A form has fields neither the ATS adapter nor the heuristic dictionary could map.
- The user has configured an LLM provider in settings.
- The user has not exceeded a self-imposed daily call cap (default: 20/day).

If any of these is false, unmapped fields are simply skipped (highlighted red as "needs review").

## What is sent

Only field metadata. No profile values. No resume. No URL. No company name.

```json
{
  "task": "field_mapping",
  "profile_keys": [
    "identity.firstName",
    "identity.lastName",
    "identity.email",
    "compensation.noticePeriodWeeks",
    "workAuth.requiresSponsorship",
    "skills.technical[name].yearsExperience",
    "..."
  ],
  "fields": [
    {
      "id": "fld_0",
      "label": "Total years of professional experience",
      "type": "text",
      "name": "yoe",
      "options": null
    },
    {
      "id": "fld_1",
      "label": "Are you legally entitled to work in Canada?",
      "type": "radio",
      "options": ["Yes", "No"]
    }
  ]
}
```

## What is returned (model-instructed JSON)

```json
{
  "mappings": [
    {
      "id": "fld_0",
      "profileKey": "experience.totalYears",
      "confidence": "high"
    },
    {
      "id": "fld_1",
      "profileKey": "workAuth.canada",
      "confidence": "high",
      "valueTransform": "citizen|permanent|work_permit -> Yes; none -> No"
    }
  ]
}
```

The `valueTransform` field is a hint the worker uses to translate the stored profile value (e.g. `"citizen"`) into the form's expected option (`"Yes"`). The transform itself is applied in code, not by another LLM call.

## Provider choice

Two viable options. Pick one.

| | Anthropic Claude Haiku | Gemini 2.5 Flash |
|---|---|---|
| Cost at 2-3 apps/week | A few cents/month at most | $0 (free tier) |
| JSON mode reliability | Excellent | Excellent |
| Latency | Sub-second typical | Sub-second typical |
| Privacy posture | Good — Anthropic doesn't train on API data by default | Free tier may be used for training; check current terms before trusting |
| Effort to integrate | Low | Low |

**Recommendation:** Start with Gemini 2.5 Flash free tier for the learning project. If the privacy of training-on-free-tier-data concerns you (it should — confirm current Google terms before committing), switch to Claude Haiku and pay the cents.

The integration is abstracted behind a `LlmProvider` interface so swapping is one file:

```ts
interface LlmProvider {
  classifyFields(req: FieldMappingRequest): Promise<FieldMappingResponse>
}
```

## Privacy guarantees and their limits

| Claim | Holds because |
|---|---|
| Profile values never sent | Only `profile_keys` (the *names*) and field metadata are in the payload |
| Form URL not sent | Worker strips it before constructing the request |
| Company name not sent | Same |
| API key not exposed to web pages | Stored in service worker; content scripts never see it |

What this does **not** protect against:

- The LLM provider sees field labels from your applications. If you applied to "Senior MLE at OpenAI," they don't see that — but they see "Total years of PyTorch experience." Aggregated across many forms, this is low-signal.
- A compromised extension build could exfiltrate anything. Mitigation: build from your own source, don't publish to the Chrome Web Store.

## Encryption details (profile + API key)

```
passphrase  ──PBKDF2(SHA-256, 600k iter, salt)──►  256-bit key
                                                      │
                                                      ▼
profile JSON ──AES-GCM(key, random 96-bit IV)──►  ciphertext + IV + tag
```

Stored as:

```json
{
  "v": 1,
  "kdf": { "alg": "PBKDF2", "hash": "SHA-256", "iter": 600000, "salt": "<base64>" },
  "enc": { "alg": "AES-GCM", "iv": "<base64>", "ct": "<base64>" }
}
```

### Session key handling

- Passphrase entered at first use of a browser session.
- Derived key cached in `chrome.storage.session` (in-memory, cleared on browser close, not synced).
- Auto-lock after configurable idle (default 30 min) — clears session key.
- Re-prompt for passphrase after lock.

### What this protects against

- Someone with read access to your `chrome.storage.local` (extension data is not encrypted by Chrome itself on disk, contrary to common belief).
- Casual inspection of the extension's storage via DevTools.

### What it does not protect against

- Malware running as your user. The decrypted profile is in service worker memory while filling.
- A weak passphrase. Use a long one. PBKDF2 600k iterations slows brute force but doesn't stop a determined attacker on a weak passphrase.
- Browser sync (we never opt into `chrome.storage.sync` — local only).

## Daily call cap and observability

- Cap: 20 LLM calls/day, configurable.
- Counter in `chrome.storage.local`, resets at local midnight.
- Popup shows: today's calls, remaining, last call timestamp.
- All calls logged (request hash, response, ts) to a rolling 7-day debug log so you can see what the LLM was asked. Useful for the learning aspect.
