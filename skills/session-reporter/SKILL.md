---
name: session-reporter
description: "Run at the end of every SolidWorks session. Analyzes what was built, asks the user for consent, and POSTs a structured feedback payload to the knowledge base. Must run before giving the final response in any session where SolidWorks work occurred."
---

# Session Reporter

Collect and submit knowledge from this SolidWorks session to the knowledge base.
Run this after modeling is complete, before your final response.

KB_HOST: `http://192.168.40.221:8100` (or SW_KB_HOST if configured)
Use `curl` via Bash for all requests. Never use Fetch or WebFetch.

---

## Step 1 — Detect relevance

Did any SolidWorks work happen in this session?
This includes: part/assembly modeling, API calls, macro generation, design
decisions, error debugging, tolerances, fits, materials.

If nothing SolidWorks-related happened → skip silently. Do not ask the user anything.

---

## Step 2 — Look up the part

Extract the part name/number from the conversation.

```bash
curl -s "http://192.168.40.221:8100/api/parts?q={part_name}"
```

If a match is found: save its `id` as `partId`. Otherwise `partId = null`.

---

## Step 3 — Extract knowledge from the conversation

Read the full conversation and extract:

### `issues` (required — narrative string)
2–5 sentences covering: what was built, what approach Claude took, what went
wrong and had to be corrected, final state (complete / in-progress / failed).

### `instructions` (array — omit if nothing was built step-by-step)
One object per build task:
```json
{
  "content": "## How to build [part]\n\n**Material:** ...\n\n### Steps\n1. ...\n2. ...",
  "partId": "<partId or null>"
}
```
Write CORRECTED steps only. Include exact API calls and parameter values.

### `macros` (array — omit if no code was written)
```json
{
  "name": "snake_case_name",
  "language": "python | vba | swapi",
  "code": "full final working source",
  "swFeaturesUsed": ["InsertHelix", "FeatureCut4"],
  "parameters": { "diameter_mm": 12 },
  "isTemplate": false,
  "partId": "<partId or null>"
}
```
Only include complete, working final versions. Skip broken/unfinished code.

### `knownErrors` (array — omit if no concrete errors)
```json
{
  "title": "short label",
  "description": "what failed exactly",
  "swFeature": "the SW API method",
  "resolution": "exact fix that worked",
  "isResolved": true,
  "severity": "low | medium | high | critical"
}
```
Only include errors where the cause and resolution are known.

### `lessons` (array — omit if nothing non-obvious was learned)
```json
{
  "category": "modeling/API | assembly/API | tolerance | fastener | dfm | workflow",
  "title": "short rule title",
  "whatHappened": "narrative",
  "rootCause": "underlying reason",
  "prevention": "concrete actionable rule",
  "severity": "low | medium | high | critical"
}
```

---

## Step 4 — Ask for consent

Ask the user **exactly this one question**:

> "Share this session's SolidWorks data with the knowledge base to help improve future designs? (yes / no)"

- **no** (or no reply): stop here. Send nothing.
- **yes**: continue to Step 5.

---

## Step 5 — Build and POST the payload

```bash
curl -s -X POST "http://192.168.40.221:8100/api/feedback" \
  -H "Content-Type: application/json" \
  -d '{
    "issues": "<narrative>",
    "partId": "<partId or null>",
    "instructions": [...],
    "macros": [...],
    "knownErrors": [...],
    "lessons": [...]
  }'
```

Omit any array key that has no items. `issues` is always required.

Expected response: HTTP 201 with `{ "id": "...", "state": "pending" }`.

Retry up to 3 times on 5xx or timeout. Drop immediately on 4xx.
On success or final failure: continue to your final response. Never block the user.

---

## Judgment rules

| Include | Skip |
|---------|------|
| Errors with exact failure / None return / wrong output | Vague "something didn't work" |
| Complete working macros | Partial or broken code |
| Steps with exact API signatures and real values | Generic "open SolidWorks and create a part" |
| Mistakes that took more than one attempt to fix | Immediate typo fixes |

When in doubt: include it. Admin reviews before publishing.
