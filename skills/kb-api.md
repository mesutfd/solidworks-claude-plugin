---
name: kb-api
description: >
  API reference for the SolidWorks knowledge base. Tells Claude when and how
  to query the public catalog before designing a part — parts, instructions,
  macros, known errors, and lessons. No auth required for reads.
  Use this skill BEFORE starting any SolidWorks design work.
---

# SolidWorks Knowledge Base — API Reference

The knowledge base exposes a public read catalog. Query it **before** starting
any design to find existing instructions, macros, and known errors for the part.

Base URL: read from `SW_KB_HOST` environment variable (default: `http://localhost:8000`)

All GET endpoints are public — no Authorization header needed.

---

## When to query (mandatory)

| Situation | What to query |
|-----------|---------------|
| User asks to build a known part (e.g. "Shaft-M-245") | `GET /api/parts?q=Shaft-M-245` → then `GET /api/parts/{id}` |
| Starting any SolidWorks modeling session | `GET /api/errors` — check known errors to avoid |
| Need to write a macro | `GET /api/macros` — check for existing templates first |
| Uncertain about build approach | `GET /api/lessons` — check published lessons |
| Looking for step-by-step guidance | `GET /api/instructions` or `GET /api/parts/{id}/instructions` |

**Always query first. Never start designing from scratch if the KB has the answer.**

---

## Endpoints

### Find a part

```
GET {SW_KB_HOST}/api/parts?q={search_term}
GET {SW_KB_HOST}/api/parts?q={search_term}&categoryId={id}&tags[]={tag}
```

Query params (all optional):
- `q` — full-text search on part number, name, description
- `categoryId` — filter by category UUID
- `tags[]` — filter by tag (repeatable)
- `page` (default: 1), `pageSize` (default: 20)

Response: `PartListResponse`
```json
{
  "items": [
    {
      "id": "uuid",
      "partNumber": "Shaft-M-245",
      "name": "Drive Shaft M245",
      "categoryId": "uuid",
      "description": "string or null",
      "material": "Plain Carbon Steel",
      "tags": ["rotating", "transmission"],
      "status": "active",
      "createdAt": "datetime",
      "updatedAt": "datetime"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

---

### Get full part detail (with all knowledge)

```
GET {SW_KB_HOST}/api/parts/{part_id}
```

Returns `PartDetail` — the part plus ALL its linked published knowledge:
```json
{
  "id": "uuid",
  "partNumber": "Shaft-M-245",
  "name": "...",
  "material": "...",
  "instructions": [ { "id": "...", "content": "markdown text", ... } ],
  "macros":       [ { "id": "...", "name": "...", "language": "python", "code": "...", ... } ],
  "knownErrors":  [ { "id": "...", "title": "...", "description": "...", "resolution": "...", ... } ],
  "lessons":      [ { "id": "...", "category": "...", "title": "...", "prevention": "...", ... } ]
}
```

**This is the primary lookup.** Get this before building any known part.

---

### List published instructions

```
GET {SW_KB_HOST}/api/instructions
GET {SW_KB_HOST}/api/parts/{part_id}/instructions
GET {SW_KB_HOST}/api/instructions/{instruction_id}
```

Response: array of `Instruction`
```json
{
  "id": "uuid",
  "content": "markdown — full build instructions with numbered steps",
  "partId": "uuid or null",
  "status": "published",
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
```

---

### List published macros

```
GET {SW_KB_HOST}/api/macros
GET {SW_KB_HOST}/api/parts/{part_id}/macros
GET {SW_KB_HOST}/api/macros/{macro_id}
```

Response: array of `Macro`
```json
{
  "id": "uuid",
  "name": "build_shaft_m245",
  "language": "python",
  "code": "full source code",
  "description": "string or null",
  "swFeaturesUsed": ["InsertHelix", "InsertCutSwept4"],
  "parameters": { "shaft_diameter_mm": 12, "thread_pitch_mm": 1.25 },
  "isTemplate": false,
  "version": "1.0.0",
  "partId": "uuid or null",
  "status": "published"
}
```

If a macro exists for the exact part: use it directly. If a template exists for
the part TYPE (e.g. `isTemplate: true` for "shaft with thread"): adapt it.

---

### List published known errors

```
GET {SW_KB_HOST}/api/errors
```

Response: array of `KnownError`
```json
{
  "id": "uuid",
  "title": "Blind cut returns None when sketch is on boundary face",
  "description": "FeatureCut4 silently returns None if the sketch plane sits exactly on the boundary face in the cut direction.",
  "errorCode": "CAD-CUT-001",
  "swFeature": "FeatureCut4",
  "resolution": "Offset sketch plane inside the material and cut toward the open side.",
  "isResolved": true,
  "severity": "medium",
  "status": "published"
}
```

**Always check this before using any SW feature.** If a known error exists for
the feature you are about to use, read the resolution before calling it.

---

### List published lessons

```
GET {SW_KB_HOST}/api/lessons
```

Response: array of `Lesson`
```json
{
  "id": "uuid",
  "category": "modeling/API",
  "title": "InsertHelix takes 10 args; Helixdef 2 = height+pitch mode",
  "whatHappened": "...",
  "rootCause": "...",
  "prevention": "Exact call: InsertHelix(Reversed, Clockwised, Tapered, Outward, Helixdef=2, Height, Pitch, Revolution, TaperAngle, StartAngle)",
  "severity": "high",
  "status": "published"
}
```

---

### List categories

```
GET {SW_KB_HOST}/api/categories
GET {SW_KB_HOST}/api/categories/{category_id}
```

---

### Health check

```
GET {SW_KB_HOST}/health
```

Returns `{ "status": "ok" }`. Call this first to confirm the server is reachable
before making knowledge lookups.

---

## Recommended pre-design sequence

```
1. GET /health                          → confirm server reachable
2. GET /api/parts?q={part_name}         → does this part exist?
   If yes:
3. GET /api/parts/{part_id}             → get full detail (instructions + macros + errors + lessons)
   Use what you find. Skip steps 4-6.
   If no:
4. GET /api/errors                      → load all known errors, check before each feature
5. GET /api/lessons                     → load relevant lessons
6. GET /api/macros                      → check for reusable templates
```

If the server is unreachable (health check fails): proceed without KB lookup.
Log the failure in the session `issues` field when submitting feedback.

---

## Error handling

| HTTP status | Meaning | Action |
|-------------|---------|--------|
| 200 / 201 | OK | Use the response |
| 404 | Not found | Part doesn't exist yet — build from scratch |
| 422 | Validation error | Check field names / types |
| 5xx / timeout | Server error | Proceed without KB, note in issues |

Do not block on a failed lookup. The KB is a helper, not a blocker.
