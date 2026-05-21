# Example: FEEDBACK

## NEEDS_REVISION

agent_editor returns structured review results with prioritized issues.

```
@AIF/2.0
FROM: agent_editor
TO: agent_searcher
TYPE: FEEDBACK
ID: F001
REF: R001
REPORT_TO: agent_editor
---
VERDICT: NEEDS_REVISION
SCORE: 78/100
ISSUES:
  - ID:I01  SEVERITY:HIGH
    LOCATION: "items #4, #7"
    DESC: "published_at field missing"
    FIX: "Add ISO-8601 timestamp to all items"
  - ID:I02  SEVERITY:MED
    LOCATION: "items #2, #9"
    DESC: "Summary exceeds 30-word limit"
    FIX: "Trim to ≤30 words"
APPROVED_ITEMS:
  - "Source attribution present on all items"
  - "Category coverage meets spec (tech/business/world)"
REVISE_FOCUS: [I01]
OPTIONAL_FIX: [I02]
```

## APPROVED

```
@AIF/2.0
FROM: agent_reviewer
TO: agent_rd
TYPE: FEEDBACK
ID: F002
REF: R002
REPORT_TO: agent_reviewer
---
VERDICT: APPROVED
SCORE: 95/100
APPROVED_ITEMS:
  - "Token expiry handled correctly with 401"
  - "No secrets appear in log output"
  - "Input validated before decode"
SUGGEST:
  - ID:S01
    DESC: "Consider adding token refresh endpoint"
```
