---
name: project-memory
description: "Short-term and long-term memory management for AI agents — works with all models. Use this skill whenever an agent needs to persist session context, remember facts/entities across sessions, or manage conversation history. Triggers: 'remember this', 'save to memory', 'what do you remember', 'recall', 'forget', 'persist state', 'maintain context', agent memory setup, cross-session memory. Always use this skill at the start of a new agent session or when the agent needs to store information for future use. Activates automatically if a .agents/ directory exists in CWD."
version: 1.0.1
author: project-memory-skill
---

# Project Memory Skill

Manage short-term (STM) and long-term (LTM) memory for AI agents via JSON files in `.agents/memory/` at CWD.

---

## Storage Layout

```
.agents/
└── memory/
    ├── stm.json          # active session working memory
    ├── ltm.json          # persistent facts, entities, preferences
    └── episodes/
        └── YYYYMMDD_HHMMSS.json   # archived sessions
```

Auto-create `.agents/memory/episodes` if they don't exist:
```bash
mkdir -p .agents/memory/episodes
```

---

## STM — Short-Term Memory

File: `.agents/memory/stm.json`

**Schema:**
```json
{
  "session_id": "YYYYMMDD_HHMMSS",
  "started_at": "ISO8601",
  "working": {
    "task": "current task description",
    "step": "current step",
    "scratch": {}
  },
  "messages": [
    { "role": "user|assistant|tool", "content": "...", "ts": "ISO8601" }
  ]
}
```

**Rules:**
- **Session start** → read `stm.json`. If `session_id` differs from now → archive first, then reset.
- **Each turn** → update `working`, append to `messages`.
- **Session end** → archive to `episodes/YYYYMMDD_HHMMSS.json`, then clear `stm.json`.
- **Trim** → if `messages` exceeds 20 items, summarize older turns into `working.scratch` and drop them.

---

## LTM — Long-Term Memory

File: `.agents/memory/ltm.json`

**Schema:**
```json
{
  "user": {
    "name": { "value": "...", "updated_at": "ISO8601", "previous_value": null },
    "preferences": { "value": {}, "updated_at": "ISO8601", "previous_value": null }
  },
  "project": {
    "name": { "value": "...", "updated_at": "ISO8601", "previous_value": null },
    "context": { "value": "...", "updated_at": "ISO8601", "previous_value": null },
    "goals": { "value": [], "updated_at": "ISO8601", "previous_value": null }
  },
  "entities": {
    "<entity_name>": { "value": {}, "updated_at": "ISO8601", "previous_value": null }
  },
  "facts": {
    "<key>": { "value": "...", "updated_at": "ISO8601", "previous_value": null }
  }
}
```

**Rules:**
- **Before responding** → read `ltm.json`, inject relevant context into reasoning.
- **Update** → overwrite `value`, move old value to `previous_value`, set `updated_at` to now.
- **Delete** → set `value` to `null`, preserve `previous_value`.
- **Add categories freely** — add new root keys as needed (`team`, `system`, etc).
- **Conflict** → LTM wins over STM unless user explicitly provides a new value.

---

## Episodes — Session Archive

Folder: `.agents/memory/episodes/`

One file = one archived session (STM snapshot at session end).

**Retrieval rules:**
- Read filenames to identify session timestamps.
- Load a specific episode only when user explicitly requests recall or context demands it.
- Never load all episodes at once.

---

## Lifecycle

```
SESSION START
  └─ Read stm.json
       ├─ session_id matches current? → continue
       └─ session_id differs or empty?
            ├─ Archive stm.json → episodes/<session_id>.json
            └─ Reset stm.json with new session_id

EACH TURN
  ├─ Read ltm.json → inject relevant context
  ├─ Update stm.working + stm.messages
  └─ Write ltm.json if new/changed info detected

SESSION END
  ├─ Archive stm.json → episodes/YYYYMMDD_HHMMSS.json
  └─ Clear stm.json
```

---

## Bash Helpers

```bash
# Read STM
cat .agents/memory/stm.json 2>/dev/null || echo "{}"

# Read LTM
cat .agents/memory/ltm.json 2>/dev/null || echo "{}"

# Archive STM
SESSION=$(python3 -c "import sys,json; d=json.load(open('.agents/memory/stm.json')); print(d.get('session_id','unknown'))")
cp .agents/memory/stm.json ".agents/memory/episodes/${SESSION}.json"

# Reset STM with new session
python3 -c "
import json
from datetime import datetime, timezone
sid = datetime.now().strftime('%Y%m%d_%H%M%S')
ts = datetime.now(timezone.utc).isoformat()
json.dump({'session_id': sid, 'started_at': ts, 'working': {}, 'messages': []}, open('.agents/memory/stm.json','w'), indent=2)
"
```

---

## Agent Tips

- **Store selectively** — only persist what the agent genuinely needs across sessions.
- **LTM is source of truth** — STM is scratch space; LTM is durable state.
- **Trim aggressively** — STM is not a full log. Summarize when `messages` > 20.
- **Lazy episode loading** — only fetch past episodes when explicitly needed.
