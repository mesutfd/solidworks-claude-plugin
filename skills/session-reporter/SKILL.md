---
name: session-reporter
description: "Run at the end of every SolidWorks session. Collects all generated knowledge and code, checks the 'always-send' preference, then uses AskUserQuestion to present an interactive consent prompt — separate from and after the task output. POSTs to /api/feedback on approval."
---

# Session Reporter

## Step 1 — Detect relevance

Did any SolidWorks work happen in this session?
(modeling, API calls, macro/code generation, design decisions, error debugging)

If nothing SolidWorks-related happened → stop silently. Do not ask anything.

---

## Step 2 — Check "always send" preference

```bash
PREF=$(cat ~/.sw-feedback-pref 2>/dev/null || echo "")
echo "$PREF"
```

If output is `always` → skip Steps 3–5 (no consent question), jump to Step 6 (POST).

---

## Step 3 — Look up the part

Extract the part name/number from the conversation.

```bash
curl -s "http://192.168.40.221:8100/api/parts?q={part_name}"
```

Save the matching `id` as `partId`. Use `null` if nothing found.

---

## Step 4 — Extract ALL knowledge and code from the conversation

### `issues` (required)
2–5 sentences: what was built, approach, what went wrong and was fixed, final state.

### `instructions`
Corrected steps only. Include exact SolidWorks API calls and real parameter values.
```json
{ "content": "## How to build [part]\n\n### Steps\n1. ...", "partId": "<uuid or null>" }
```

### `macros` — ALL code generated in this session

**Capture EVERY code block written or shared with the user:**
- Python scripts for SolidWorks COM automation
- VBA macros
- SWAPI call sequences
- Helper scripts, test code, any code shown in a code block
- Adapted templates

One entry per code artifact:
```json
{
  "name": "snake_case_name",
  "description": "one sentence: what this code does",
  "language": "python | vba | swapi",
  "code": "<FULL VERBATIM SOURCE — complete, never summarized or truncated>",
  "swFeaturesUsed": ["InsertHelix", "FeatureCut4"],
  "parameters": { "shaft_diameter_mm": 12 },
  "isTemplate": false,
  "partId": "<uuid or null>"
}
```

The `code` field MUST contain the complete source as shown to the user.
Never omit code because it's short or a snippet. Every code block goes here.

### `knownErrors`
```json
{
  "title": "short label",
  "description": "what failed exactly — include the return value or error message",
  "swFeature": "SW API method",
  "resolution": "exact fix that worked",
  "isResolved": true,
  "severity": "low | medium | high | critical"
}
```
Only include errors where cause AND resolution are both known.

### `lessons`
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

## Step 5 — Ask consent via AskUserQuestion (interactive prompt)

Use the **AskUserQuestion tool** to present the consent prompt. This creates a
proper interactive UI element that is separate from and appears after the task
output — the user must click an option before you continue.

Call AskUserQuestion with:
```json
{
  "questions": [
    {
      "question": "Share this session's SolidWorks knowledge with the knowledge base?",
      "header": "KB Feedback",
      "multiSelect": false,
      "options": [
        {
          "label": "Yes, send now",
          "description": "Submit instructions, macros, errors, and lessons collected this session (pending admin review)"
        },
        {
          "label": "Always send",
          "description": "Send now and auto-submit in all future sessions without asking again"
        },
        {
          "label": "Skip this session",
          "description": "Do not submit feedback for this session"
        }
      ]
    }
  ]
}
```

Wait for the user's selection before proceeding.

- **"Yes, send now"** → continue to Step 6 (POST)
- **"Always send"** → continue to Step 6 (POST), then Step 7 (save preference)
- **"Skip this session"** → stop here, do not POST

---

## Step 6 — POST the payload

Use Python to handle multiline code and special characters safely:

```bash
python3 << 'PYEOF'
import subprocess, json, sys

SESSION_ID = "<SESSION_ID from session context>"
KB_HOST = "http://192.168.40.221:8100"

payload = {
    "issues": """<issues text>""",
    "sessionId": SESSION_ID,
    "instructions": [
        {"content": """<instruction content>""", "partId": None}
    ],
    "macros": [
        {
            "name": "script_name",
            "description": "what it does",
            "language": "python",
            "code": """<FULL SOURCE CODE>""",
            "swFeaturesUsed": ["FeatureExtrusion3"],
            "parameters": {},
            "isTemplate": False,
            "partId": None
        }
    ],
    "knownErrors": [
        {
            "title": "error title",
            "description": "what failed",
            "swFeature": "APIMethodName",
            "resolution": "fix applied",
            "isResolved": True,
            "severity": "medium"
        }
    ],
    "lessons": [
        {
            "category": "modeling/API",
            "title": "lesson title",
            "whatHappened": "...",
            "rootCause": "...",
            "prevention": "...",
            "severity": "medium"
        }
    ]
}

# Remove null partId at top level
if payload.get("partId") is None:
    payload.pop("partId", None)

# Remove empty arrays
for key in ["instructions", "macros", "knownErrors", "lessons"]:
    if not payload.get(key):
        payload.pop(key, None)

body = json.dumps(payload)

for attempt in range(3):
    result = subprocess.run(
        ["curl", "-s", "-w", "\n%{http_code}", "-X", "POST",
         f"{KB_HOST}/api/feedback",
         "-H", "Content-Type: application/json",
         "-d", body],
        capture_output=True, text=True
    )
    output = result.stdout.strip().split("\n")
    http_code = output[-1] if output else "0"
    body_out = "\n".join(output[:-1])
    
    if http_code.startswith("2"):
        print(f"Feedback submitted. ID: {json.loads(body_out).get('id', '?')}")
        break
    elif http_code.startswith("4"):
        print(f"Feedback rejected ({http_code}): {body_out}")
        break
    else:
        print(f"Attempt {attempt+1} failed ({http_code}). Retrying...")

PYEOF
```

---

## Step 7 — Save "always" preference (only if user chose "Always send")

```bash
echo "always" > ~/.sw-feedback-pref
echo "Auto-send preference saved."
```

---

## Judgment rules

| Include | Skip |
|---------|------|
| ALL code blocks written in this session | Nothing — every code artifact goes in |
| Errors with exact failure / return value / error message | Vague "something didn't work" |
| Steps with exact SW API calls and real parameter values | Generic "open SolidWorks and create a part" |
| Final corrected version of each code artifact | Earlier broken/intermediate versions |
| Non-obvious patterns and lessons from both success and failure | Common knowledge |
