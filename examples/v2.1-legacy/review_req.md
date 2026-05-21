# Example: REVIEW_REQ

## ATTACH: REF (content stored in a prior DELIVER)

agent_searcher asks agent_editor to review the search results without re-sending the full payload.

```
@AIF/2.0
FROM: agent_searcher
TO: agent_editor
TYPE: REVIEW_REQ
ID: R001
REF: T001
REPORT_TO: agent_searcher
TRUNCATE: true
---
ARTIFACTS: [news_results_D001]
ATTACH: REF
CONTENT_REF: D001
ACCEPT:
  - A01: "Minimum 10 items with all fields present"
  - A02: "All items within past 24 hours"
NOTE: "Please verify factual completeness and format compliance"
```

`TRUNCATE: true` tells the receiver not to forward the upstream payload downstream.  
`CONTENT_REF: D001` points to the DELIVER message that contains the actual content.

## ATTACH: INLINE (content embedded directly)

```
@AIF/2.0
FROM: agent_rd
TO: agent_reviewer
TYPE: REVIEW_REQ
ID: R002
REF: T003
REPORT_TO: agent_rd
---
ARTIFACTS: [auth.py]
ATTACH: INLINE
ACCEPT:
  - A01: "Handles expired tokens with 401"
  - A02: "No secrets logged"
RESULT:
  auth.py: |
    def verify_token(token: str) -> dict:
        payload = jwt.decode(token, SECRET, algorithms=["HS256"])
        return payload
NOTE: "Focus on security — this is auth-critical"
```
