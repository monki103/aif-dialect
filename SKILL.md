# AIF Dialect Skill

AIF (Agent Interchange Format) is the standard communication format used between agents in a multi-agent system.
（AIF 是多 Agent 系統中 Agent 之間的標準溝通格式。）
All agent-to-agent (M2M) communication uses this format instead of natural language — it improves efficiency, precision, and reduces output token usage by replacing natural language decoration with structured key-value bodies.

AIF is domain-agnostic: it applies to any multi-agent task (research, writing, data analysis, code review, etc.), not just software development.

---

## When to Use AIF
（使用時機）

Use AIF **only** when communication involves another agent — not when replying directly to a human user.

### USE AIF — any one condition is enough
```
- Another agent is the receiver (TO: agent_*)
- Task involves review, QA, or approval by a separate agent
- Task requires back-and-forth collaboration between agents
- Task is delegated to a sub-agent (multi-step, multi-role pipeline)
- User explicitly mentions an agent role: @agent-pm / @agent-rd / etc.
```

### USE NLU — reply in natural language instead
```
- The receiver is a human user (TO: USER, or REPORT_TO: USER is triggered)
- Single Q&A, factual lookup, or clarification directed at the user
- Casual conversation with no delegation needed
- Task is fully answerable without involving another agent
```

> **Rule**: When `TO` is `USER` (or the final reply goes to a human), always translate to natural language. Never send raw AIF to a user.

---

### `<aif>` Wrapper Tag & 3-Layer Fallback Extraction
（`<aif>` 包裝標籤與 3 層 Fallback 解析規則）

Wrap AIF messages in `<aif>...</aif>` for reliable extraction from LLM output:

```xml
<aif>
@AIF/2.0
FROM: agent_rd
TO: agent_pm
TYPE: DELIVER
ID: D001
REF: T001
REPORT_TO: agent_pm
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

**Request → Response pair:**

```
# M1 → M2
<aif>
@AIF/2.0
FROM: agent_pm
TO: agent_rd
TYPE: TASK
ID: T001
REF: -
REPORT_TO: agent_pm
---
GOAL: "Summarize today's top 3 AI research papers"
PRIORITY: HIGH
</aif>

# M2 → M1  (correct — AIF only, no NLU decoration)
<aif>
@AIF/2.0
FROM: agent_rd
TO: agent_pm
TYPE: DELIVER
ID: D001
REF: T001
REPORT_TO: agent_pm
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
@AIF/2.0
FROM: <sender>
TO: <receiver>
TYPE: <message_type>
ID: <unique_id>
REF: <referenced_id or ->
REPORT_TO: <agent_name or USER>
TRUNCATE: true | false          ← optional: strip upstream payload; downstream uses CONTENT_REF instead
SKILLS: [<skill_name>, ...]     ← optional: extra skills the sub-agent should load
MODEL: <model_hint>             ← optional: preferred model capability for this task
---
[BODY]
```

### Header Fields
（Header 欄位說明）

| Field | Required | Description |
|-------|----------|-------------|
| `FROM` | ✅ | Name of the sending agent（發送方）|
| `TO` | ✅ | Name of the receiving agent（接收方）|
| `TYPE` | ✅ | Message type (see list below)（訊息類型）|
| `ID` | ✅ | Unique message ID（唯一 ID）|
| `REF` | ✅ | ID of the message being referenced; use `-` if none（參照 ID）|
| `REPORT_TO` | ✅ | Who to report to after the task is done — agent name or `USER`（回報對象）|
| `TRUNCATE` | ❌ | When `true`, receiver must NOT relay upstream payload to downstream agents; use `CONTENT_REF` instead. Default: `false`（是否截斷上游 payload 的轉發）|
| `SKILLS` | ❌ | Extra skills the receiving agent should load for this task（選填）|
| `MODEL` | ❌ | Preferred model capability hint for the receiving agent: `standard` \| `pro` \| `opus`（選填）|

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
# Rule 1: Single value
KEY: VALUE

# Rule 2: List of items
ISSUES:
  - ID:I01  SEVERITY:HIGH
    LOCATION: "item #3"
    DESC: "Publish time missing from search result"
    FIX: "Add published_at field to all items"

# Rule 3: Grouped fields
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
MODE: FULL | COMPACT            ← optional: COMPACT inherits SPEC from upstream TASK（選填）
INHERITS: <task_id>            ← optional: inherit full SPEC from referenced TASK（MODE=COMPACT 時使用）
DELTA:                         ← optional: overrides/additions on top of INHERITS（選填）
  PRIORITY: HIGH
PARALLEL_TASKS: [<id>, ...]    ← optional（選填）
CONTEXT_REF: <parent_task_id>  ← optional（選填）
SUGGESTED_ROUTE: LOCAL | T0 | T3 | T4  ← optional: routing hint — LOCAL(<100ms local) / T0(cloud top inference) / T3(local inference) / T4(system instruction)（選填）
```

### DELIVER
```
STATUS: COMPLETED | PARTIAL | FAILED
SUMMARY: "<result summary — high level only, do not duplicate full content here>"
ARTIFACTS: [<item1>, <item2>, ...]  ← optional: list of deliverable names/IDs（選填）
RESULT:                              ← optional: structured output, only if not inline（選填）
  KEY: VALUE
ISSUES_RESOLVED: [<id>, ...]        ← optional（選填）
NOTE: "<additional notes>"          ← optional（選填）
NOISE_RATIO: <float 0.0–1.0>       ← optional: ratio of unstructured noise in LLM output（選填）
```

### REVIEW_REQ
```
# ← AIF/2.0 Breaking: v1 field DELIVER renamed to ARTIFACTS
ARTIFACTS: [<item1>, <item2>, ...]   ← items to review (documents, data, outputs, files)
ATTACH: INLINE | REF                ← INLINE = content included in body; REF = reference only
CONTENT_REF: <deliver_id>           ← required when ATTACH=REF: ID of the DELIVER containing content
ACCEPT:                             ← optional: acceptance criteria from original TASK
  - A01: "<criterion>"
CONTEXT_REF: <task_id>             ← optional: reference to original TASK
NOTE: "<notes>"                    ← optional
```

### FEEDBACK
```
VERDICT: APPROVED | NEEDS_REVISION | REJECTED
SCORE: <n>/100                      ← optional（選填）
ISSUES:
# ← AIF/2.0 Breaking: FILE/LINE removed; use LOCATION instead
  - ID:<id>  SEVERITY:CRITICAL|HIGH|MED|LOW
    LOCATION: "<optional: where in the output>"   ← optional, e.g. item #3, section 2, row 7
    DESC: "<issue description>"
    FIX: "<suggested fix>"
SUGGEST:
  - ID:<id>
    LOCATION: "<optional>"
    DESC: "<suggestion>"
APPROVED_ITEMS:
  - "<description of what passed>"
REVISE_FOCUS: [<issue_id>, ...]     ← optional: top-priority issues to fix（選填）
OPTIONAL_FIX: [<issue_id>, ...]     ← optional: LOW severity items, not required（選填）
NOISE_RATIO: <float 0.0–1.0>       ← optional: ratio of unstructured noise in LLM output（選填）
```

### QUERY
```
QUESTIONS:
  - Q01: "<question>"
  - Q02: "<question>"
CONTEXT: "<background info>"        ← optional（選填）
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

## Severity Levels
（Severity 等級）

```
CRITICAL  →  Output is unusable or factually incorrect（輸出不可用或有嚴重錯誤）
HIGH      →  Key requirement not met, must fix before delivery（關鍵需求未達成，必須修正）
MED       →  Affects quality, should fix（影響品質，建議修正）
LOW       →  Enhancement suggestion, optional（優化建議，可選修正）
```

---

## Full Examples
（完整範例）

Scenario: agent_pm dispatches a news search task, agent_searcher delivers results,
agent_editor reviews and requests revision.

### TASK
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

### REVIEW_REQ
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

### FEEDBACK
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

### TASK (COMPACT mode — inheriting upstream spec)
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
