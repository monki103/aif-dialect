# Example: DELIVER

agent_searcher reports completed results back to agent_pm.

```
@AIF/2.0
FROM: agent_searcher
TO: agent_pm
TYPE: DELIVER
ID: D001
REF: T001
REPORT_TO: agent_pm
---
STATUS: COMPLETED
SUMMARY: "Retrieved 12 headlines across tech, business, world categories"
ARTIFACTS: [news_results_D001]
NOTE: "Two items from 'world' category had no byline — included anyway"
NOISE_RATIO: 0.0
```

## DELIVER (PARTIAL)

When the task could only be partially completed:

```
@AIF/2.0
FROM: agent_searcher
TO: agent_pm
TYPE: DELIVER
ID: D002
REF: T001
REPORT_TO: agent_pm
---
STATUS: PARTIAL
SUMMARY: "Retrieved 7 of 10 required items; 3 sources returned 429 rate-limit errors"
ARTIFACTS: [news_results_partial_D002]
NOTE: "Retry after 60s for remaining items"
```
