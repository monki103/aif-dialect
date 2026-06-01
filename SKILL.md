# AIF Dialect Skill (v3.0)

AIF v3 is a **machine-to-machine (M2M) dialect** for hub-spoke agent
orchestration. A single main-agent orchestrates N sessionless sub-agents
via a file-backed message log (`handoff_book.md`). All inter-agent
traffic is structured JSON; no natural-language prose ever travels
between agents.

Non-goal: AIF is a transport protocol, not a replacement for model
semantic understanding. Do not treat model-internal state (for example
KV cache) as an inter-agent message format.

This skill is loaded **only by the main-agent**. Sub-agents receive a
small inline stub (defined in §9) at spawn time and do not load this
file. For the design rationale and history of v3, see
`DESIGN.md`. v2.1 spec is in git history (commit `b2f9cf3`
and earlier).

---

## 1. Goal: M2M Strict

Messages between agents are structured data, not natural-language prose:

- No problem-restatement, polite preamble, emotive filler, or explain-back
- Bodies are JSON; no Markdown narrative inside a body
- Only the `user ↔ main-agent` boundary uses natural language;
  `agent ↔ agent` never does

If a sub-agent emits prose outside the AIF envelope, or returns a body
that is not valid JSON, the main-agent MUST reject the reply and either
re-spawn with a stricter prompt or treat the spawn as `FAILED`.

---

## 2. Topology

```
   USER ◀── NLU ──▶ MAIN-AGENT ◀── AIF ──▶ SUB-AGENT(s)
                       │
                       ▼
               handoff_book.md   (file-backed; main owns)
```

| Component         | Loads SKILL.md    | File I/O           | Spawns sub | Session    |
|-------------------|-------------------|--------------------|------------|------------|
| `main-agent`      | yes               | yes (sole writer)  | yes        | stateful   |
| `sub-agent`       | no (stub only)    | no                 | no         | sessionless|
| `handoff_book.md` | —                 | append-only        | —          | —          |

**Asymmetry is structural, not optional**: sub-agents are pure functions
spawned by main; they have no persistent state, no file access, and do
not spawn downstream sub-agents.

---

## 3. Message Envelope

Every AIF message appended to `handoff_book.md` has this exact shape:

````
# Title: <single-line subject>;
## Description: <optional context, or 'non'>;
*TO:* <role>@<model>;
*FROM:* <role>[@<model>];
*TYPE:* TASK | DELIVER | QUERY | CLARIFY | SUMMARY;
*ID:* YYYYMMDD-HHMMSS-NN;
*IN-REPLY-TO:* <prior ID>;
*MESSAGE:*
```json
{ ... single JSON object ... }
```
===END_OF_MESSAGE===
````

### Header rules

- Every header line except `*MESSAGE:*` ends with `;`
- `# Title` (Markdown H1, 1 line); `## Description` (Markdown H2, may be `non`)
- `*TO:*` is single-valued; multi-recipient dispatch splits into N TASKs
- `*FROM:*`: `main-agent` may omit `@model`
- `*ID:*` format `YYYYMMDD-HHMMSS-NN`, where `NN` is a sub-second counter
  `00`–`99` to avoid collisions when main spawns multiple sub-agents
  within the same second
- `*IN-REPLY-TO:*` is **omitted** for: the initial TASK, fan-in spawns
  (use `inputs:` in body instead), and SUMMARY messages.
  It is **required** for: every DELIVER, every QUERY, every CLARIFY.

### Body rules

- Body is exactly one fenced JSON code block (` ```json ... ``` `)
  immediately after `*MESSAGE:*`
- JSON object, snake_case keys
- No prose outside or inside the fence
- The message ends with `===END_OF_MESSAGE===` on its own line after
  the closing fence

---

## 4. TYPE Semantics

| TYPE      | Direction      | Purpose                                  |
|-----------|----------------|------------------------------------------|
| `TASK`    | main → sub     | Spawn a sub to perform work              |
| `DELIVER` | sub → main     | Sub returns work result (incl. failure)  |
| `QUERY`   | sub → main     | Sub asks for clarification mid-task      |
| `CLARIFY` | main → sub     | Main answers a QUERY                     |
| `SUMMARY` | main → main    | Main checkpoints state from log entries  |

These 5 are the entire v3 Core. `REVIEW_REQ` / `FEEDBACK` / `REVISE` /
`ACK` / `CANCEL` from v2.1 are application-layer concerns and encoded as
`TASK` with appropriate body shape — they are not protocol TYPEs in v3.

---

## 5. Body Schemas

All schemas are required-fields contracts. Producers MUST include every
required field; consumers MUST treat unknown extra fields as opaque.

### 5.1 TASK

```json
{
  "goal": "<one-line objective>",
  "deliverable": {
    "format": "<expected output shape name>",
    "required_fields": ["<field>", "..."],
    "acceptance": "<criteria, string or object>"
  },
  "inputs": ["<msg_id>", "..."],
  "scope": ["<bullet>", "..."],
  "constraints": ["<bullet>", "..."]
}
```

Required: `goal`, `deliverable`.
`inputs` is required when the TASK forwards or fans-in prior messages.
`scope` and `constraints` are optional.

### 5.2 DELIVER

```json
{
  "status": "COMPLETED",
  "summary": "<one line>",
  "result": { "...": "shape determined by triggering TASK.deliverable.format" },
  "notes": "<string or object>"
}
```

Required: `status`, `summary`, `result`.
`status` is one of `"COMPLETED"`, `"PARTIAL"`, `"FAILED"`.
`notes` is optional.

### 5.3 QUERY

```json
{
  "question": "<one-line>",
  "blocking": true,
  "fields": ["<missing_field>", "..."],
  "context_needed": ["<bullet>", "..."]
}
```

Required: `question`, `blocking`.
`blocking: true` halts the sub until CLARIFY arrives. `blocking: false`
lets the sub proceed with a best-guess and surface the assumption in
its eventual DELIVER `notes`.
Header `*IN-REPLY-TO:*` MUST point to the TASK the sub is processing.

### 5.4 CLARIFY

```json
{
  "answer": { "...": "..." },
  "notes": "<optional string>"
}
```

Required: `answer`. When the originating QUERY listed `fields`, the
`answer` object SHOULD use the same keys.
Header `*IN-REPLY-TO:*` MUST point to the QUERY's ID.

### 5.5 SUMMARY

```json
{
  "source_ids": ["<msg_id>", "..."],
  "summary": { "...": "structured key-value content" }
}
```

Required: `source_ids`, `summary`.
`summary` MUST be a JSON object, not a free-form string.
SUMMARY messages have `*FROM:*` = `*TO:*` = `main-agent` and no
`*IN-REPLY-TO:*`.

---

## 6. Logging Invariant

**Every AIF message MUST be appended to `handoff_book.md`. No exceptions.**

This includes:

- Every TASK at sub-agent spawn (including pure forwards — body
  `inputs` lists the forwarded IDs)
- Every DELIVER, QUERY, CLARIFY
- Every SUMMARY main produces

Consequences:

- `handoff_book.md` is the single source of truth and audit log
- main MAY discard verbatim message content from its working memory
  and re-read by ID on demand
- main MAY self-checkpoint by writing a SUMMARY, resetting its own
  context, and reloading from that SUMMARY

Violating this invariant breaks both self-recovery and audit chain.

---

## 7. Reply Chain Rules

- `*IN-REPLY-TO:*` carries the **conversational link** (single value)
- `inputs` in TASK body carries **fan-in / forward references** (array)
- When both apply (e.g. fixer TASK after review), `*IN-REPLY-TO:*`
  points to the **primary triggering message** and `inputs` enumerates
  any additional referenced IDs

**QUERY/CLARIFY side-branch rule**: when a sub asks a QUERY and main
returns a CLARIFY, the sub's eventual DELIVER's `*IN-REPLY-TO:*` MUST
still point to the **original TASK**, NOT to the CLARIFY. The
QUERY↔CLARIFY pair is a side-branch; the TASK→DELIVER main chain is
unaffected.

---

## 8. Main-Agent Responsibilities

1. **Sole writer** of `handoff_book.md`. Sub-agents never read or write
   the file directly.
2. **ID generation**: produce `YYYYMMDD-HHMMSS-NN` from wall-clock time
   and a sub-second counter. Increment `NN` for same-second siblings.
3. **Context assignment at spawn**: inline relevant prior messages
   (or a SUMMARY of them) into the sub's launch prompt. The sub does
   not see `handoff_book.md` directly.
4. **Reject ill-formed replies**: if a sub's reply contains prose
   outside the AIF envelope, or the body is not valid JSON conforming
   to §5, main MUST reject and either re-spawn with a stricter prompt
   or record the spawn as `FAILED` via a synthesised DELIVER.
5. **Two-tier context management** (routine, not emergency):
   - **Soft forget** (per cycle): after each sub reply, drop verbatim
     content from main's working memory; keep only IDs and a 1-line
     summary in main's working state
   - **Hard forget** (at checkpoint): when main's own context nears
     the model's limit, write a SUMMARY message, reset main's context,
     reload from the SUMMARY
6. **Maintain working state** separate from `handoff_book.md`:

   ```
   current_goal: <user request, 1 line>
   current_step: <e.g. awaiting reviewer DELIVER>
   recent_ids: [<id>, ...]
   project_state: <3–5 line rolling summary>
   open_questions: []
   ```

---

## 9. Sub-Agent Stub

When main spawns a sub-agent, it MUST inline this template (or a
semantically equivalent one) into the sub's launch prompt. The sub does
not load SKILL.md.

````
You are receiving an AIF v3 TASK from main-agent. Reply with a single
AIF DELIVER message in exactly the format shown below. Output ONLY
the DELIVER. No prose before or after.

=== INCOMING TASK ===
<paste full AIF TASK message: header + JSON body + ===END_OF_MESSAGE===>

=== REPLY FORMAT (mandatory) ===

# Title: <one-line>;
## Description: non;
*TO:* main-agent;
*FROM:* <your_role>@<your_model>;
*TYPE:* DELIVER;
*ID:* YYYYMMDD-HHMMSS-NN;
*IN-REPLY-TO:* <task_id>;
*MESSAGE:*
```json
{
  "status": "COMPLETED" | "PARTIAL" | "FAILED",
  "summary": "<one line>",
  "result": { ... shape per task.deliverable.format ... },
  "notes": "<optional>"
}
```
===END_OF_MESSAGE===

If you cannot complete the TASK without more information, instead
reply with a QUERY (same envelope, *TYPE:* QUERY, body per spec).
Main will answer with CLARIFY before you resume.
````

Sub-agent context cost: ≈300 tokens for the stub above plus the inlined
TASK and any inlined prior messages main provides.

---

## 10. Minimum Working Example

````
# Title: Plan project: directory-watcher CLI;
## Description: non;
*TO:* agent-pm@opus-4.7;
*FROM:* main-agent;
*TYPE:* TASK;
*ID:* 20260521-090000-00;
*MESSAGE:*
```json
{
  "goal": "A Python CLI that watches a directory and POSTs file-change events to a webhook URL.",
  "deliverable": {
    "format": "milestone-list",
    "required_fields": ["id", "name", "depends_on", "est_effort_hours", "acceptance_criteria"]
  }
}
```
===END_OF_MESSAGE===

# Title: Milestone plan for directory-watcher CLI;
## Description: non;
*TO:* main-agent;
*FROM:* agent-pm@opus-4.7;
*TYPE:* DELIVER;
*ID:* 20260521-090145-00;
*IN-REPLY-TO:* 20260521-090000-00;
*MESSAGE:*
```json
{
  "status": "COMPLETED",
  "summary": "5 milestones; 2 parallel streams",
  "result": {
    "milestones": [
      {"id": "M1", "name": "skeleton", "depends_on": [], "est_effort_hours": 1, "acceptance_criteria": "pipx install + --help"}
    ]
  }
}
```
===END_OF_MESSAGE===
````

For full end-to-end scenarios — including QUERY/CLARIFY, review, and
fixer cycles — see `examples/v3-walkthrough/handoff_book.md` and
`examples/v3-walkthrough/query_clarify.md`.

---

## 11. Conformance

A v3 conformant implementation satisfies all of the following:

1. Every AIF message written to `handoff_book.md` matches the §3 envelope
2. Every body parses as JSON and matches the §5 schema for its declared TYPE
3. The producer is `main-agent` if and only if it has read/write access
   to `handoff_book.md`
4. Reply chains follow §7 (single-value `IN-REPLY-TO` for the primary
   link, `inputs` for fan-in, QUERY/CLARIFY as side-branch)
5. No natural-language prose appears in any agent-to-agent message —
   inside or outside the body

---

## 12. Out of Scope

- User↔main-agent dialogue (NLU; not AIF's concern)
- Agent role definitions and orchestration patterns (application layer)
- The decision of when to invoke AIF vs another protocol (application)
- v2.1 `REVIEW_REQ` / `FEEDBACK` / `REVISE` / `ACK` / `CANCEL` types
  (encode as `TASK` with appropriate body)
- Sub-agent self-spawn (not supported; only main spawns)
- Cross-`handoff_book.md` communication (each main-agent owns one log)

---

Spec version: **3.0**
