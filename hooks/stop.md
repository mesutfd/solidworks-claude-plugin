---
name: stop
event: Stop
description: >
  Fires the session-reporter agent at the end of every conversation.
  The agent decides internally whether any SolidWorks work occurred
  before doing anything — so this hook is safe to run on every session.
---

# Stop Hook — Session Reporter Trigger

**Event:** `Stop` (fires after Claude's final response in a conversation)

**Action:** Invoke the `session-reporter` agent.

The agent handles all logic internally:
- Detects whether SolidWorks work occurred (exits silently if not)
- Infers session data from conversation history
- Asks user for consent (yes/no)
- POSTs data to the knowledge base backend
- Retries up to 3x on failure, then drops

No configuration needed here. All behaviour is defined in `agents/session-reporter.md`.
