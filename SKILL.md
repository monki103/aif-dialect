# AIF Dialect Skill (v2.1)

AIF (Agent Interchange Format) is a **machine-to-machine (M2M) dialect** for agents sharing an LLM substrate. It specifies how agent-to-agent messages are structured, identified, and cross-referenced.

This skill is the **AIF Core** layer: role-agnostic, single-file installable, sufficient on its own to send and receive valid AIF messages. For the design rationale and invariants, see `DESIGN.md`. For concrete transport implementations, see `profiles/`.

What AIF does **not** cover: when to engage AIF vs natural language, who the agents are, how user-facing translation works. Those belong to the application layer.

---

## Self-Activation
（自動觸發）

This skill is **self-activating** — if it is present in an agent's loaded context, the agent uses AIF for M2M traffic; otherwise the agent falls back to natural language.

**Architectural rule for upstream components:**

- Slash commands, role definitions, and orchestrators **MUST NOT** hardcode `Load: aif-dialect` into sub-agent prompts.
- They MUST describe workflows in **protocol-agnostic** terms — `TASK` / `DELIVER` / `REVIEW_REQ` / `FEEDBACK` / `REVISE` / `QUERY` / `CLARIFY` / `ACK` / `CANCEL` are generic multi-agent message-type names, not AIF-specific.
- Whether those messages serialize as AIF or NLU is decided here at runtime by this skill's presence.
- If this skill is not loaded in the receiver's context, the sender MUST fall back to natural language.

---

## Default Working Language (I8)
（預設工作語言）

The default working language for all M2M traffic is **English**. This includes:

- All AIF body free-form string fields (`GOAL`, `SUMMARY`, `DESC`, `NOTE`, etc.)
- Transport envelope prose fields, where applicable (see profiles)

**Override conditions**:
- The user-task explicitly requires another working language (e.g., a localization task, authoring content in a target locale)
- The deployment has set a different M2M working language

The user-locale boundary — translating the final result back to the user's natural language — is the **application layer's** responsibility, not AIF's. AIF only stipulates the default M2M language.

Mnemonic: *every hop inside AIF is English; the application's gateway switches to user locale on the last hop out.*

---

## `<aif>` Wrapper Tag & 3-Layer Fallback Extraction
（包裝標籤與解析規則）

Wrap AIF messages in `<aif>...</aif>` for reliable extraction from LLM output:

```xml
<aif>
@AIF/2.1
FROM: agent_a
TO: agent_b
TYPE: DELIVER
ID: D001
REF: T001
REPORT_TO: agent_a
---
STATUS: COMPLETED
SUMMARY: "Task completed successfully"
</aif>
```

**3-Layer Fallback Extraction Rule:**

| Priority | Method | Condition |
|----------|--------|-----------|
| 1 | `<aif>...</aif>` tag | Tag present in LLM output |
| 2 | Fenced code block (` ``` `) | No `<aif>` tag; parse first AIF-formatted block |
| 3 | Raw text scan | No tag or block; scan for `@AIF/` prefix line |

**MUST**: When replying to an AIF message, always wrap your response in `<aif>...</aif>`. No preamble, no explanation, no natural language outside the tag.

**Skill Teaching Rule:** If the incoming message contains `@AIF/`, the sender is already AIF-capable — do NOT include the full skill spec in your reply. Respond in AIF format directly. To delegate to a downstream agent that may not know AIF, use `SKILLS: [aif-dialect]` in the header instead of inlining the spec.

**Request → Response pair:**

```
# M1 → M2
<aif>
@AIF/2.1
FROM: agent_a
TO: agent_b
TYPE: TASK
ID: T001
REF: -
REPORT_TO: agent_a
---
GOAL: "Summarize today's top 3 AI research papers"
PRIORITY: HIGH
</aif>

# M2 → M1  (correct — AIF only, no NLU decoration)
<aif>
@AIF/2.1
FROM: agent_b
TO: agent_a
TYPE: DELIVER
ID: D001
REF: T001
REPORT_TO: agent_a
---
STATUS: COMPLETED
SUMMARY: "3 papers summarized: attention mechanisms, RLHF scaling, tool-use benchmarks"
ARTIFACTS: [papers_summary]
</aif>
```

---

## Message Structure
（訊息結構）

Every AIF message has two parts: a **Header** and a **Body**, separated by `---`.

```
@AIF/2.1
FROM: <sender>
TO: <receiver>
TYPE: <message_type>
ID: <unique_id>
REF: <referenced_id or ->
REPORT_TO: <agent_name or USER>
TRUNCATE: true | false          ← optional: strip upstream payload
SKILLS: [<skill_name>, ...]     ← optional: skills the receiver should load
MODEL: <tier>                   ← optional: preferred capability tier
---
[BODY]
```

### Header Fields

| Field | Required | Description |
|-------|----------|-------------|
| `FROM` | ✅ | Sending agent name（發送方）|
| `TO` | ✅ | Receiving agent name（接收方）|
| `TYPE` | ✅ | Message type (see list below)（訊息類型）|
| `ID` | ✅ | Unique message ID; concrete format depends on transport（唯一 ID）|
| `REF` | ✅ | ID of the referenced message; use `-` if none（參照 ID）|
| `REPORT_TO` | ✅ | Who to report to after the task — agent name or `USER`（回報對象）|
| `TRUNCATE` | ❌ | When `true`, receiver MUST NOT relay upstream payload to downstream; use a context-handoff policy instead. Default: `false` |
| `SKILLS` | ❌ | Skills the receiver should load. Application-influenced. |
| `MODEL` | ❌ | Preferred capability tier: `standard | pro | opus`. Application-influenced. |

---

## Message Types (TYPE)
（訊息類型）

| TYPE | Purpose | Expected Reply |
|------|---------|----------------|
| `TASK` | Assign a task（指派任務）| `DELIVER` |
| `DELIVER` | Report task result（回報結果）| none (or `ACK`) |
| `REVIEW_REQ` | Request a review（請求審查）| `FEEDBACK` |
| `FEEDBACK` | Reply with review results（回覆審查）| `REVISE` or none |
| `REVISE` | Resubmit after making changes（修改後重新提交）| `FEEDBACK` |
| `QUERY` | Ask a question or request clarification（詢問）| `CLARIFY` |
| `CLARIFY` | Answer a question（回答）| none |
| `ACK` | Confirm receipt（確認收到）| none |
| `CANCEL` | Cancel a task（取消任務）| `ACK` |

---

## Body Format Rules
（Body 格式規則）

```
# Single value
KEY: VALUE

# List of items
ISSUES:
  - ID:I01  SEVERITY:HIGH
    LOCATION: "item #3"
    DESC: "Publish time missing from search result"
    FIX: "Add published_at field to all items"

# Grouped fields
SPEC:
  FEATURES:
    - F01: "Retrieve top 10 news from past 24 hours"
    - F02: "Cover tech, business, world categories"
  CONSTRAINTS:
    - C01: "Sources must be reputable major outlets"
    - C02: "Summaries must not exceed 30 words"
```

---

## Body Schema by TYPE
（各 TYPE 的 Body 格式）

### TASK

```
GOAL: "<task objective>"
SPEC:
  FEATURES:
    - F01: "<feature description>"
  CONSTRAINTS:
    - C01: "<constraint>"
ACCEPT:
  - A01: "<acceptance criteria>"
PRIORITY: HIGH | MED | LOW
MODE: FULL | COMPACT             ← optional: COMPACT inherits SPEC from upstream
INHERITS: <task_id>              ← required when MODE=COMPACT
DELTA:                           ← optional: overrides/additions on top of INHERITS
  PRIORITY: HIGH
PARALLEL_TASKS: [<id>, ...]      ← optional
CONTEXT_REF: [<id>, ...]         ← optional: by-reference context handoff
READ_MODE: REQUIRED | OPTIONAL   ← optional: qualifies CONTEXT_REF (default REQUIRED)
CONTEXT_SUMMARY: "<digest>"      ← optional: by-summary context handoff
CONTEXT_SOURCE: [<id>, ...]      ← optional: source IDs that CONTEXT_SUMMARY was derived from
SUGGESTED_ROUTE: LOCAL | T0 | T3 | T4  ← optional: routing hint
```

See **Context Handoff Policies** (below) for `CONTEXT_REF` / `CONTEXT_SUMMARY` / `READ_MODE` semantics.

### DELIVER

```
STATUS: COMPLETED | PARTIAL | FAILED
SUMMARY: "<result summary — high level only, do not duplicate full content here>"
ARTIFACTS: [<item1>, <item2>, ...]  ← optional: list of deliverable names/IDs
RESULT:                              ← optional: structured output, only if not inline
  KEY: VALUE
ISSUES_RESOLVED: [<id>, ...]        ← optional
NOTE: "<additional notes>"          ← optional
NOISE_RATIO: <float 0.0–1.0>       ← optional: unstructured-noise ratio of LLM output
```

### REVIEW_REQ

```
ARTIFACTS: [<item1>, <item2>, ...]   ← items to review (documents, data, outputs, files)
ATTACH: INLINE | REF                 ← INLINE = content embedded in body; REF = reference only
CONTENT_REF: <deliver_id>            ← required when ATTACH=REF
ACCEPT:                              ← optional: acceptance criteria from original TASK
  - A01: "<criterion>"
CONTEXT_REF: [<id>, ...]             ← optional: by-reference handoff for the reviewer
READ_MODE: REQUIRED | OPTIONAL       ← optional
NOTE: "<notes>"                      ← optional
```

### FEEDBACK

```
VERDICT: APPROVED | NEEDS_REVISION | REJECTED
SCORE: <n>/100                       ← optional
ISSUES:
  - ID:<id>  SEVERITY:CRITICAL|HIGH|MED|LOW
    LOCATION: "<where in the output>"   ← optional
    DESC: "<issue description>"
    FIX: "<suggested fix>"
SUGGEST:
  - ID:<id>
    LOCATION: "<optional>"
    DESC: "<suggestion>"
APPROVED_ITEMS:
  - "<description of what passed>"
REVISE_FOCUS: [<issue_id>, ...]      ← optional
OPTIONAL_FIX: [<issue_id>, ...]      ← optional
NOISE_RATIO: <float 0.0–1.0>        ← optional
```

### REVISE

```
SUMMARY: "<what was changed>"
ARTIFACTS: [<item1>, ...]            ← updated deliverables
ISSUES_RESOLVED: [<issue_id>, ...]
NOTE: "<additional context>"         ← optional
```

### QUERY

```
QUESTIONS:
  - Q01: "<question>"
  - Q02: "<question>"
CONTEXT: "<background info>"         ← optional
```

### CLARIFY

```
ANSWERS:
  - Q01: "<answer>"
  - Q02: "<answer>"
```

### ACK

```
STATUS: RECEIVED
```

### CANCEL

```
REASON: "<reason for cancellation>"
```

---

## Context Handoff Policies (I7)
（上下文交接政策）

Sub-agents are typically single-shot — a newly dispatched agent starts with no prior context. AIF offers two policies for transferring upstream context to a downstream agent, plus a combined mode.

### By-reference

Sender provides IDs; receiver fetches the originals via `RESOLVE(id)` (see Transport Abstraction).

```
CONTEXT_REF: [20260521-142500-03, 20260520-091500-02]
READ_MODE: REQUIRED   ← receiver MUST read all referenced messages (default)
```

Or advisory:

```
CONTEXT_REF: [20260521-142500-03]
READ_MODE: OPTIONAL   ← receiver MAY skip if not relevant to the task
```

**Trade-off**: high fidelity (originals preserved), more tokens (receiver reads them all).

### By-summary

Sender digests upstream context inline; source IDs are preserved for audit.

```
CONTEXT_SUMMARY: "Upstream chose option A after rejecting option B on perf grounds. Constraints inherited: ALLOWED paths in src/auth/."
CONTEXT_SOURCE: [20260521-142500-03, 20260520-091500-02]
```

**Trade-off**: low token cost (digest only), but lossy. The source IDs let the receiver verify if needed.

### Combined

Both policies on the same TASK — receiver gets a fast path (summary) and a verification path (source IDs):

```
CONTEXT_SUMMARY: "..."
CONTEXT_SOURCE: [<id>, ...]
CONTEXT_REF: [<id>, ...]       ← optional additional refs not summarized
READ_MODE: OPTIONAL
```

---

## Transport Abstraction (I6)
（傳輸抽象）

AIF messages are addressable by ID. The sender and receiver share a **transport** that provides:

```
RESOLVE(id) → message
```

Given an ID, the transport returns the full message (header + body). Concrete implementations live in `profiles/`; the contract itself is core.

### ID requirements (core)

- Every message MUST carry a unique `ID` within the transport's scope.
- IDs MUST be addressable: given an ID, the transport's `RESOLVE` returns the message, or signals "not found".
- The concrete ID format is defined by the transport profile. Example: `file-backed-transport` uses `YYYYMMDD-HHMMSS-NN`.

### Transport profiles

- `profiles/file-backed-transport.md` — append-only log file. `RESOLVE(id)` is an `awk` range extract over the log file.
- (Future) in-memory transport, broker-based transport (Redis / SQS / NATS).

If no transport profile is active, IDs remain meaningful within a single session but `RESOLVE` is unavailable — context-handoff policies that require RESOLVE (by-reference) cannot be used in that deployment.

---

## Severity Levels
（嚴重度等級）

```
CRITICAL  →  Output is unusable or factually incorrect
HIGH      →  Key requirement not met, must fix before delivery
MED       →  Affects quality, should fix
LOW       →  Enhancement suggestion, optional
```

---

## Full Examples
（完整範例）

Examples below use generic placeholders (`agent_a`, `agent_b`, `agent_c`) to underline role independence — substitute any names that fit the application.

### TASK

```
@AIF/2.1
FROM: agent_a
TO: agent_b
TYPE: TASK
ID: T001
REF: -
REPORT_TO: agent_a
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

### TASK with by-summary context handoff

```
@AIF/2.1
FROM: agent_a
TO: agent_c
TYPE: TASK
ID: T002
REF: -
REPORT_TO: agent_a
---
GOAL: "Format prior search results into a daily newsletter"
CONTEXT_SUMMARY: "agent_b delivered 12 news items in 3 categories (tech/business/world). All items have valid published_at within 24h. Two world-category items lacked byline. Source: D001 in transport log."
CONTEXT_SOURCE: [D001]
ACCEPT:
  - A03: "Newsletter must have a title and date header"
  - A04: "Items grouped by category"
PRIORITY: MED
```

### REVIEW_REQ with TRUNCATE + CONTENT_REF

```
@AIF/2.1
FROM: agent_b
TO: agent_c
TYPE: REVIEW_REQ
ID: R001
REF: T001
REPORT_TO: agent_b
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

### FEEDBACK

```
@AIF/2.1
FROM: agent_c
TO: agent_b
TYPE: FEEDBACK
ID: F001
REF: R001
REPORT_TO: agent_c
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

### TASK (COMPACT mode — inheriting upstream spec)

```
@AIF/2.1
FROM: agent_a
TO: agent_c
TYPE: TASK
ID: T003
REF: T001
REPORT_TO: agent_a
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

---

## Forward Compatibility
（向前相容）

- Receivers MUST ignore unknown fields. This allows additive evolution of the spec.
- Receivers MUST reject messages missing required header fields.
- Experimental fields SHOULD be prefixed with `X_` (e.g., `X_PRIORITY_REASON`) to signal non-standard usage.

## Changes from v2.0
（v2.0 → v2.1 變更）

**Added**:
- *Default Working Language* section (formalizes I8: M2M defaults to English)
- *Context Handoff Policies* section: by-reference (existing `CONTEXT_REF` + new `READ_MODE`) and by-summary (new `CONTEXT_SUMMARY` + `CONTEXT_SOURCE`)
- *Transport Abstraction* section: `RESOLVE(id)` contract, ID requirements, pointer to profiles
- `REVISE` body schema (was implicit in v2.0)

**Removed** (moved to application layer):
- "When to Use AIF" section — application policy, not a protocol concern. See `skills/universal-agent/SKILL.md` for the language-gateway pattern.

**Compatibility**:
- v2.0 messages remain valid under v2.1 (new fields are optional and additive).
- v2.1 senders targeting v2.0 receivers should avoid the new fields (`CONTEXT_SUMMARY`, `CONTEXT_SOURCE`, `READ_MODE`).
- The `examples/` directory retains v2.0-shaped examples; they remain valid demonstrations of the schema.
