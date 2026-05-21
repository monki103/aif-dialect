# Example: TASK

agent_pm delegates a news search task to agent_searcher.

```
@AIF/2.0
FROM: agent_pm
TO: agent_searcher
TYPE: TASK
ID: T001
REF: -
REPORT_TO: agent_pm
---
GOAL: "Search today's top headlines across major news categories"
SPEC:
  FEATURES:
    - F01: "Retrieve top 10 news items from past 24 hours"
    - F02: "Cover categories: tech, business, world"
    - F03: "Each item: title, source, published_at, summary (≤30 words)"
  CONSTRAINTS:
    - C01: "Sources must be reputable major outlets"
    - C02: "Summaries must be factual, no editorializing"
ACCEPT:
  - A01: "Minimum 10 items with all fields present"
  - A02: "All items within past 24 hours"
PRIORITY: HIGH
```

## TASK (COMPACT mode)

agent_pm assigns a follow-up task that inherits the spec from T001.

```
@AIF/2.0
FROM: agent_pm
TO: agent_editor
TYPE: TASK
ID: T002
REF: T001
REPORT_TO: agent_pm
---
GOAL: "Format search results into a daily newsletter"
MODE: COMPACT
INHERITS: T001
DELTA:
  PRIORITY: MED
  ADDITIONAL_ACCEPT:
    - A03: "Newsletter must have a title and date header"
    - A04: "Items grouped by category"
```

COMPACT mode avoids duplicating the full SPEC from T001 — the receiver reads it from the referenced task.
