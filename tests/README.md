# AIF Conformance Tests

These tests define the expected behavior of any AIF-compatible parser or agent implementation.

## Structure

```
tests/
├── valid/      ← messages that MUST be accepted and parsed without error
└── invalid/    ← messages that MUST be rejected or flagged as non-conforming
```

## How to Use

### Manual (LLM-based)

Load [SKILL.md](../SKILL.md) into a session, then present each test file and ask:

- For `valid/`: "Is this a valid AIF message? Parse and describe it."  
  Expected: parsed successfully, all required fields identified.

- For `invalid/`: "Is this a valid AIF message?"  
  Expected: rejected, with the specific violation named.

### Automated

A conformance runner can:

1. Send each `valid/*.aif` to an agent and assert no parse errors
2. Send each `invalid/*.aif` and assert a rejection reason matching the filename

## Valid Test Cases

| File | What it tests |
|------|--------------|
| `task_full.aif` | TASK with all optional fields |
| `task_minimal.aif` | TASK with required fields only |
| `deliver_completed.aif` | DELIVER STATUS:COMPLETED |
| `deliver_partial.aif` | DELIVER STATUS:PARTIAL |
| `review_req_inline.aif` | REVIEW_REQ with ATTACH:INLINE |
| `review_req_ref.aif` | REVIEW_REQ with ATTACH:REF + CONTENT_REF |
| `feedback_approved.aif` | FEEDBACK VERDICT:APPROVED |
| `feedback_needs_revision.aif` | FEEDBACK VERDICT:NEEDS_REVISION with ISSUES |
| `query.aif` | QUERY with multiple questions |
| `clarify.aif` | CLARIFY answering a QUERY |
| `ack.aif` | ACK minimal |
| `cancel.aif` | CANCEL with reason |

## Invalid Test Cases

| File | Violation |
|------|-----------|
| `missing_from.aif` | FROM field absent |
| `missing_ref.aif` | REF field absent |
| `missing_header_sep.aif` | No `---` separator between header and body |
| `missing_version.aif` | No `@AIF/` version line |
| `unknown_type.aif` | TYPE value not in the allowed set |
| `to_user_aif_body.aif` | TO:USER but body is AIF format (must be NLU) |
| `review_req_ref_no_content_ref.aif` | ATTACH:REF without CONTENT_REF |
| `feedback_no_verdict.aif` | FEEDBACK missing required VERDICT |
