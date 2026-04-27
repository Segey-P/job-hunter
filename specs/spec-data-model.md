# 02 — Data Model

The profile is a single JSON object. Markdown is for *editing convenience* via the options page, not the storage format. JSON is the source of truth.

## Profile schema (v1)

```json
{
  "schemaVersion": 1,
  "identity": {
    "firstName": "",
    "lastName": "",
    "preferredName": "",
    "email": "",
    "phone": "",
    "phoneCountryCode": "+1"
  },
  "address": {
    "street": "",
    "city": "Squamish",
    "region": "BC",
    "postalCode": "",
    "country": "Canada"
  },
  "links": {
    "linkedin": "",
    "github": "",
    "portfolio": "",
    "other": []
  },
  "workAuth": {
    "canada": "citizen",
    "usa": "none",
    "uk": "none",
    "eu": "none",
    "requiresSponsorship": false
  },
  "compensation": {
    "currency": "CAD",
    "expectedMin": null,
    "expectedMax": null,
    "noticePeriodWeeks": 0,
    "willingToRelocate": false,
    "willingToTravel": "occasional"
  },
  "demographics": {
    "gender": "decline",
    "ethnicity": "decline",
    "veteran": "decline",
    "disability": "decline"
  },
  "experience": {
    "totalYears": 0,
    "roles": [
      {
        "company": "",
        "title": "",
        "startDate": "YYYY-MM",
        "endDate": "YYYY-MM | present",
        "summary": ""
      }
    ]
  },
  "education": [
    {
      "institution": "",
      "degree": "",
      "field": "",
      "startYear": null,
      "endYear": null
    }
  ],
  "skills": {
    "technical": [
      { "name": "Python", "yearsExperience": 0, "level": "advanced" }
    ],
    "domain": [
      { "name": "Banking / financial services", "yearsExperience": 0 }
    ],
    "languages": [
      { "name": "English", "proficiency": "native" }
    ]
  },
  "standardAnswers": {
    "whyThisCompany": "",
    "whyThisRole": "",
    "greatestStrength": "",
    "biggestWeakness": "",
    "fiveYearPlan": ""
  }
}
```

## Schema design notes

- **`schemaVersion`** is mandatory. Migrations will be needed; this is non-negotiable.
- **`workAuth` per region** — common screening question is "are you authorized to work in X without sponsorship." Boolean per country is more flexible than free text.
- **Demographics defaults to `"decline"`** — these are voluntary disclosure fields. Default should never silently disclose. User can change per-field.
- **`compensation.noticePeriodWeeks`** — nearly every Canadian/UK application asks. Numeric beats free text.
- **`skills.technical[].yearsExperience`** — the most common screening question is "Years of X." This needs to be queryable per skill, not a free-form list.
- **No resume blob** — resume is uploaded manually. Browser security prevents pre-attaching files via extension. Don't pretend otherwise.

## What's deliberately not in the schema

| Not included | Why |
|---|---|
| Resume PDF/binary | Can't pre-attach via extension; manual upload step |
| Cover letter content | Too contextual; tailoring is a separate (later) feature |
| References | Almost never asked at apply-time |
| Salary history | Illegal to ask in many jurisdictions; not worth storing |
| Date of birth | Never legitimately needed at apply-time in CA/US |
| SIN/SSN | Never at apply-time; if asked, that's a red flag |

## Application tracker schema

```json
{
  "applications": [
    {
      "id": "uuid",
      "ts": "ISO8601",
      "company": "",
      "role": "",
      "url": "",
      "atsType": "greenhouse | lever | ashby | workday | icims | unknown",
      "fieldsFilled": 0,
      "fieldsSkipped": 0,
      "profileSchemaVersion": 1,
      "notes": ""
    }
  ]
}
```

`profileSchemaVersion` lets you reason about "what version of my profile did I send?" if you change it later.

## Storage at rest

| Key | Encrypted? | Contents |
|---|---|---|
| `profile` | yes (AES-GCM) | Profile JSON |
| `apiKey` | yes (AES-GCM) | LLM provider API key |
| `applications` | no | Tracker (no PII beyond what you already chose to apply with) |
| `settings` | no | LLM provider choice, UI preferences |
| `kdfParams` | no | Salt + iteration count for PBKDF2 |

Encryption details in `04-llm-integration.md` (it covers key handling).

## Editing UX

The options page renders the schema as a structured form (sections matching the JSON top-level keys). Optional toggle: "Edit as raw JSON" for power users — useful for the schema fields that are arrays of objects (`experience.roles`, `skills.technical`).

A separate "Import from Markdown" feature is **not** in v1. If wanted later, it would be a one-way parser that fills the form, not a round-tripping format.
