# GitHub Copilot Instructions — aif-dialect

## What This Repo Is

`aif-dialect` is the public specification for **AIF (Agent Interchange Format) 2.0** — a model-agnostic structured message format for agent-to-agent (M2M) communication. It contains no runnable code; the deliverables are:

- `SKILL.md` — the canonical spec (also works as a Claude Code slash command)
- `examples/` — annotated reference messages per TYPE
- `tests/` — conformance test cases (`.aif` files) for valid/invalid messages

There is no build system, package manager, or test runner. Conformance tests are validated manually by loading `SKILL.md` into an LLM session.

---

## Architecture

```
aif-dialect/
├── SKILL.md              ← Canonical spec + Claude Code skill loader
├── examples/             ← One .md per message TYPE with annotated examples
│   ├── task.md           ← TASK + COMPACT mode
│   ├── deliver.md        ← DELIVER COMPLETED / PARTIAL
│   ├── review_req.md     ← ATTACH:INLINE vs ATTACH:REF
│   ├── feedback.md       ← FEEDBACK APPROVED / NEEDS_REVISION
│   └── query_clarify.md  ← QUERY + CLARIFY pair
└── tests/
    ├── README.md         ← How to run conformance tests
    ├── valid/            ← 12 messages that MUST parse correctly
    └── invalid/          ← 8 messages that MUST be rejected
```

`SKILL.md` is the single source of truth. `README.md` is a bilingual (EN + 繁體中文) summary with research data. `HANDOFF.md` contains design decisions and known next steps.

---

## AIF Message Format

Every AIF message:

1. Starts with `@AIF/2.0` on the first line
2. Contains required header fields: `FROM`, `TO`, `TYPE`, `ID`, `REF`, `REPORT_TO`
3. Uses `---` as the separator between header and body
4. Is wrapped in `<aif>...</aif>` tags in LLM output

**Required header fields:**

| Field | Notes |
|-------|-------|
| `FROM` | Sender agent name |
| `TO` | Receiver agent name |
| `TYPE` | One of: `TASK`, `DELIVER`, `REVIEW_REQ`, `FEEDBACK`, `REVISE`, `QUERY`, `CLARIFY`, `ACK`, `CANCEL` |
| `ID` | Unique message ID |
| `REF` | ID of referenced message; `-` if none |
| `REPORT_TO` | Agent name or `USER` |

**Optional header fields:** `TRUNCATE`, `SKILLS`, `MODEL`

---

## Key Conventions

### SKILL.md is the spec, not documentation
Treat `SKILL.md` as a machine-readable protocol spec. It defines MUST rules and few-shot enforcement examples. Don't add explanatory prose — keep it prescriptive.

### `<aif>` wrapper is mandatory for LLM output
When an LLM replies to an AIF message, the response MUST be wrapped in `<aif>...</aif>`. No preamble or natural language outside the tag.

### Skill Teaching Rule
If the incoming message already has `@AIF/2.0`, the sender is AIF-capable — do NOT re-include the full spec. Use `SKILLS: [aif-dialect]` in the outgoing header instead of inlining spec content.

### `TRUNCATE: true` prevents context explosion
When a message has `TRUNCATE: true`, the receiver MUST NOT relay upstream payload downstream. Use `CONTENT_REF: <deliver_id>` in `REVIEW_REQ` to reference content without duplicating it.

### ACCEPT field is the quality driver
The `ACCEPT` field in TASK messages defines explicit acceptance criteria. Always include it — it is the primary mechanism that improves output quality over NLU.

### AIF → Human translation rule
When `REPORT_TO: USER` resolves (i.e., the final reply goes to a human), translate to natural language. Never expose raw AIF to a user.

### `tests/invalid/` filenames encode the violation
Each invalid test file is named after the rule it violates (e.g., `missing_from.aif`, `unknown_type.aif`). When adding a new invalid test, the filename must match the violation name.

### FEEDBACK body structure
`ISSUES` entries use inline ID + SEVERITY on the same line:
```
- ID:I01  SEVERITY:HIGH
  LOCATION: "..."
  DESC: "..."
  FIX: "..."
```
`FILE`/`LINE` fields were removed in AIF/2.0; use `LOCATION` instead.

---

## Conformance Testing

**Manual (LLM-based):**
1. Load `SKILL.md` into a session
2. For `tests/valid/*.aif`: ask "Is this a valid AIF message? Parse and describe it." — expect successful parse
3. For `tests/invalid/*.aif`: ask "Is this a valid AIF message?" — expect rejection with the specific violation named

No automated runner exists yet (tracked in `HANDOFF.md` as a known next step).

---

## Relationship to `agent-skills`

`agent-skills` (`yihsin428/agent-skills`) is the upstream experiment harness. `SKILL.md` here is the stable public version. To sync from upstream:

```bash
cp /path/to/agent-skills/skills/aif-dialect/SKILL.md ./SKILL.md
```
