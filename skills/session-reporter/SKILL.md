---
name: session-reporter
description: "Run at the end of every SolidWorks session. Collects all generated knowledge and code, checks 'always-send' preference, asks interactive consent, and POSTs the structured payload to the knowledge base. Operates in two phases across two turns."
---

# Session Reporter

## Architecture — Two-Phase Interactive Flow

This skill spans two conversation turns by design:

**Phase A** (end of the modeling turn — when you invoke this skill):
- Check auto-send preference. If "always" → build payload, POST immediately, done in one turn.
- Otherwise: extract all knowledge and code, write a pending marker to disk,
  display the payload summary, and end the response with ONLY the consent question.

**Phase B** (start of the NEXT turn — handled by the hook):
- The hook instructs you to check for the pending marker at the start of every turn.
- The user's answer is processed before anything else in that turn.

**This is what makes it feel interactive and blocking:**
the response does not conclude with a task summary — it concludes with the consent question,
and nothing else happens until the user answers it.

---

## Step 1 — Detect relevance

Did any SolidWorks work happen in this session?
This includes: part/assembly modeling, API calls, macro generation, code written,
design decisions, error debugging, tolerances, fits, materials.

If nothing SolidWorks-related happened → skip silently. Do not ask anything.

---

## Step 2 — Check "always send" preference

```bash
PREF=$(cat ~/.sw-feedback-pref 2>/dev/null || echo "")
echo "preference: $PREF"
```

- If output is `always` → skip Steps 4–5 (no consent question needed), go directly to Step 6 (POST).
- Otherwise → continue to Step 3.

---

## Step 3 — Look up the part

Extract the part name/number from the conversation.

```bash
curl -s "http://192.168.40.221:8100/api/parts?q={part_name}"
```

If a match is found: save its `id` as `partId`. Otherwise `partId = null`.

---

## Step 4 — Extract ALL knowledge and code from the conversation

Read the entire conversation history and extract the following.

### `issues` (required — narrative string)
2–5 sentences covering: what was built, approach taken, what went wrong and was
corrected, final state (complete / in-progress / failed).

### `instructions` (array — steps to build the part)
Write CORRECTED steps only. Include exact SolidWorks API calls and parameter values.
```json
{
  "content": "## How to build [part]\n\n**Material:** ...\n\n### Steps\n1. ...\n2. ...",
  "partId": "<partId or null>"
}
```

### `macros` (array — ALL code generated in this session)

**CRITICAL: Capture EVERY piece of code you wrote or ran during this session.**

Include in macros:
- Every Python script for SolidWorks automation (pywin32, COM, win32com)
- Every VBA macro written
- Every SWAPI call sequence
- Any helper functions, utility scripts, or code snippets shared with the user
- Templates that were adapted

For EACH code artifact, create one entry:
```json
{
  "name": "snake_case_descriptive_name",
  "description": "one sentence: what this code does",
  "language": "python | vba | swapi",
  "code": "<FULL VERBATIM SOURCE — complete, not summarized, not truncated>",
  "swFeaturesUsed": ["InsertHelix", "FeatureCut4", "FeatureExtrusion3"],
  "parameters": { "shaft_diameter_mm": 12, "thread_pitch_mm": 1.25 },
  "isTemplate": false,
  "partId": "<partId or null>"
}
```

Rules for `code` field:
- **Full source only** — never a summary, never "..." truncation
- Include imports, class definitions, helper functions — everything needed to run it
- If the code was corrected mid-session: include the FINAL working version only
- If multiple versions exist: include the final one, note corrections in `description`

Rules for `swFeaturesUsed`:
- Extract from the code: every SolidWorks API method name called (InsertHelix, FeatureCut4, etc.)
- If the code doesn't call SW APIs directly, list the SW features it automates

**Do not skip code because it's "short" or "simple". Every code artifact the user received must be in this array.**

### `knownErrors` (array — concrete failures with resolutions)
```json
{
  "title": "short label",
  "description": "what failed exactly — include the return value or error message",
  "swFeature": "the SW API method that failed",
  "resolution": "exact fix that worked",
  "isResolved": true,
  "severity": "low | medium | high | critical"
}
```
Only include errors where BOTH cause AND resolution are known.

### `lessons` (array — non-obvious rules that emerged)
```json
{
  "category": "modeling/API | assembly/API | tolerance | fastener | dfm | workflow",
  "title": "short rule title",
  "whatHappened": "narrative of what happened",
  "rootCause": "underlying reason it happened",
  "prevention": "concrete actionable rule for the future",
  "severity": "low | medium | high | critical"
}
```

---

## Step 5 — Write pending marker and display summary

Write a marker so the hook knows feedback is pending:
```bash
echo '{"pending":true}' > ~/.sw-pending-feedback.json
```

Then display the payload summary to the user. Format it clearly:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SOLIDWORKS SESSION KNOWLEDGE READY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Part:         [part name or "general session"]
 Instructions: [N] build steps
 Macros:       [N] code artifacts
               • [macro name] ([language], [N] lines)
               • ...
 Known Errors: [N] (all resolved)
 Lessons:      [N] patterns captured
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Send to knowledge base?

  yes     → send now
  no      → skip this session
  always  → send now + auto-send in all future sessions

```

**This is the LAST thing in your response. Do not add any closing text, summary,
or sign-off after this block. The response ends here.**

---

## Step 6 — POST the payload (runs after "yes" or "always" consent, or auto-runs if preference="always")

Use Python to build and send the payload to avoid shell escaping issues with code:

```bash
python3 << 'PYEOF'
import subprocess, json, os

payload = {
    "issues": """<issues narrative>""",
    "sessionId": "<SESSION_ID from session context>",
    "partId": "<partId or null — use JSON null, not the string 'null'>",
    "instructions": [
        {
            "content": """<full instruction content>""",
            "partId": "<partId or null>"
        }
    ],
    "macros": [
        {
            "name": "macro_name",
            "description": "what this does",
            "language": "python",
            "code": """<FULL SOURCE CODE verbatim>""",
            "swFeaturesUsed": ["InsertHelix"],
            "parameters": {},
            "isTemplate": False,
            "partId": None
        }
    ],
    "knownErrors": [
        {
            "title": "title",
            "description": "what failed",
            "swFeature": "FeatureCut4",
            "resolution": "exact fix",
            "isResolved": True,
            "severity": "medium"
        }
    ],
    "lessons": [
        {
            "category": "modeling/API",
            "title": "title",
            "whatHappened": "...",
            "rootCause": "...",
            "prevention": "...",
            "severity": "medium"
        }
    ]
}

# Remove null partId at top level (server expects absence, not null)
if payload.get("partId") is None:
    del payload["partId"]

# Remove empty arrays
for key in ["instructions", "macros", "knownErrors", "lessons"]:
    if not payload.get(key):
        payload.pop(key, None)

body = json.dumps(payload)
result = subprocess.run(
    ["curl", "-s", "-w", "\n%{http_code}", "-X", "POST",
     "http://192.168.40.221:8100/api/feedback",
     "-H", "Content-Type: application/json",
     "-d", body],
    capture_output=True, text=True
)
print(result.stdout)
PYEOF
```

Expected: HTTP 201 with `{ "id": "...", "state": "pending" }`.

Retry up to 3 times on 5xx or connection error. Drop on 4xx.

---

## Step 7 — Handle "always" consent: save preference

If the user answered "always":
```bash
echo "always" > ~/.sw-feedback-pref
echo "Auto-send preference saved."
```

---

## Step 8 — Clean up pending marker

```bash
rm -f ~/.sw-pending-feedback.json
```

---

## Judgment rules

| Include | Skip |
|---------|------|
| ALL code written in this session | Nothing — every code artifact goes in |
| Errors with exact failure / return value | Vague "something didn't work" |
| Steps with exact SW API calls and real parameter values | Generic "open SolidWorks and create a part" |
| Mistakes that needed more than one attempt to fix | Immediate typo fixes |
| Complete working macros | Code that was tried and abandoned (only include the final working version) |

When in doubt: include it. Admin reviews before publishing.

---

## Consent question: exact format

Always use this exact block as the final output — nothing after it:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SOLIDWORKS SESSION KNOWLEDGE READY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Part:         [name]
 Instructions: [N] steps
 Macros:       [N] — [names and languages]
 Known Errors: [N]
 Lessons:      [N]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Send to knowledge base?  →  yes / no / always
```
