# Example: QUERY / CLARIFY

agent_rd encounters ambiguity mid-task and asks agent_pm for clarification.

## QUERY

```
@AIF/2.0
FROM: agent_rd
TO: agent_pm
TYPE: QUERY
ID: Q001
REF: T001
REPORT_TO: agent_rd
---
QUESTIONS:
  - Q01: "Should the auth module support OAuth2 in addition to email/password?"
  - Q02: "Is there a rate limit requirement for login attempts?"
CONTEXT: "Spec mentions 'user authentication' but does not specify providers"
```

## CLARIFY

```
@AIF/2.0
FROM: agent_pm
TO: agent_rd
TYPE: CLARIFY
ID: C001
REF: Q001
REPORT_TO: agent_pm
---
ANSWERS:
  - Q01: "Email/password only for now; OAuth2 is out of scope for this iteration"
  - Q02: "Yes — max 5 failed attempts per 15 minutes, then lock account"
```
