# AIF Profile: File-Backed Transport

**Profile ID**: `file-backed-transport`  
**Applies to**: AIF 2.0+  
**Status**: Draft — 2026-05-21

This profile specifies how to use an append-only file as the transport layer for AIF messages. The file bus provides the envelope (addressing, audit trail, threading); AIF provides the body schema. The two layers are complementary and non-conflicting.

The **canonical reference implementation** is `backroom/handoff_book.md` governed by `backroom/handoff_book_spec.md` (pyCatchPokemon project). Other deployments may use any file path and role naming they choose.

---

## Envelope Format

Each message is a self-contained unit: header block, body, end marker.

```
Date: YYYY-MM-DD HH:MM:SS
Id: YYYYMMDD-HHMMSS-NN
In-Reply-To: <prior transport Id>
From: <sender role>
To: <recipient role>[, <recipient2>...]
Cc: <observer role>[, ...]
Title: <one-line subject>
Lang: <BCP 47 language code>
---
<body>
===END_OF_MESSAGE===
```

### Envelope Fields

| Field | Required | Rule |
|---|---|---|
| `Date` | ✅ | `YYYY-MM-DD HH:MM:SS`, 24-hour local time |
| `Id` | ✅ | `YYYYMMDD-HHMMSS-NN` (see §Transport ID Generation) |
| `In-Reply-To` | conditional | Required when replying to a specific message; omit for new threads |
| `From` | ✅ | Sender role name |
| `To` | ✅ | One or more recipient roles, comma-separated |
| `Cc` | ❌ | Observer roles, comma-separated |
| `Title` | ✅ | Single line, ≤ 80 chars; self-explanatory as a standalone summary |
| `Lang` | ✅ | BCP 47 code: `en` for M2M; user's locale for `To: USER` |

**Delimiters** (exact text, on their own line, not optional):
- `---` — separates header block from body
- `===END_OF_MESSAGE===` — terminates the message

Messages in the file are separated by one blank line.

---

## Transport ID Generation

Format: `YYYYMMDD-HHMMSS-NN`

- Time is taken at write time (local server time)
- `NN` = count of existing messages sharing the same `YYYYMMDD-HHMMSS` prefix + 1, zero-padded to two digits
- Compute `NN` before writing:
  ```bash
  grep "^Id: YYYYMMDD-HHMMSS" <log-file> | wc -l
  # result + 1 = NN; first message of that second → NN=01
  ```

---

## Append-Only Semantics

- All writes are appended to the end of the file — existing messages are never edited or deleted.
- Exception: a writer may correct a format error in a message it just wrote, before any other agent reads it.
- Protocol violations observed in another agent's message are reported by appending a new notice message, not by editing the offending entry.

---

## Body Format: AIF Inside the Envelope

For M2M messages, the body MUST be an AIF message wrapped in `<aif>...</aif>`:

```
Date: 2026-05-21 14:30:00
Id: 20260521-143000-01
In-Reply-To: 20260521-142500-03
From: Orchestrator
To: agent_pm
Lang: en
Title: Phase 8 kickoff — please plan
---
<aif>
@AIF/2.0
FROM: Orchestrator
TO: agent_pm
TYPE: TASK
ID: T001
REF: -
REPORT_TO: Orchestrator
---
GOAL: "Plan Phase 8 — performance profiling layer"
ACCEPT:
  - A01: "Module exports trace context manager"
PRIORITY: HIGH
CONTEXT_REF: 20260521-142500-03,20260520-091500-02
</aif>
===END_OF_MESSAGE===
```

For user-facing messages (`To: USER`), the body is natural language. AIF MUST NOT appear in user-facing messages (per AIF core rules).

AIF's 3-layer fallback extraction (`<aif>` tag → fenced block → raw scan) is defined in SKILL.md and applies here unchanged.

---

## Two ID Schemes

This profile introduces two independent, non-conflicting ID namespaces:

| ID | Layer | Format | Scope | Purpose |
|---|---|---|---|---|
| `Id:` (envelope) | Transport | `YYYYMMDD-HHMMSS-NN` | File-level, persistent across sessions | Audit trail, grep/awk extraction, cross-message references |
| `ID:` (AIF body) | Pipeline | `T001`, `D001`, `F001`… | Within one TASK→DELIVER chain | Pipeline-step threading |

Transport IDs are stable indefinitely; AIF IDs are scoped to the pipeline that generated them.

---

## Threading Semantics

Two threading mechanisms operate independently at different scopes:

| Field | Layer | Semantics |
|---|---|---|
| `In-Reply-To:` (envelope) | Transport | References a transport `Id:`. Tracks conversation threads in the file log. |
| `REF:` (AIF body) | Pipeline | References an AIF `ID:`. Tracks TASK→DELIVER→FEEDBACK→REVISE step ordering. |

A single message may carry both:
```
In-Reply-To: 20260521-142500-03   ← this reply continues that file-log thread
REF: T001                         ← this AIF DELIVER is in response to TASK T001
```

---

## File Log as Backing Store for `TRUNCATE` / `CONTENT_REF`

AIF 2.0 core declares `TRUNCATE: true` (don't relay upstream payload to downstream; use `CONTENT_REF` instead) but specifies no place to fetch content by ID. A file-backed transport resolves this: the log file is the backing store.

**Extended rule (profile-level)**: `CONTEXT_REF` and `CONTENT_REF` values MAY be transport IDs in addition to AIF pipeline IDs. A transport ID in either field means: "fetch this context directly from the log instead of receiving it inline."

Consumers resolve a transport ID via:
```bash
awk '/^Id: <transport-id>$/,/^===END_OF_MESSAGE===/' <log-file>
```

Example with mixed ID types:
```
CONTEXT_REF: 20260521-142500-03,20260520-091500-02   ← transport IDs → fetch from log
CONTENT_REF: D003                                     ← AIF pipeline ID → fetch from pipeline state
```

---

## `Lang:` Field

The `Lang:` envelope field is a **profile extension** — it is not part of AIF core.

| Scenario | `Lang:` value |
|---|---|
| M2M message | `en` (required; all agents operate in English regardless of user locale) |
| User-facing message (`To: USER`) | User's locale (e.g., `zh-TW`, `ja`) |
| Cc'd observer | Follows the `To:` primary recipient's language |

**`Title:` language** tracks `Lang:` — English titles for M2M, user locale for user-facing messages. Existing non-English titles in a live log are grandfathered; do not retrofit.

---

## Self-Verification (Mandatory)

After each append, the writer MUST verify all three before proceeding:

1. **Id exists and is unique**:
   ```bash
   grep -c "^Id: <new-id>" <log-file>   # must return 1
   ```
2. **Message is complete**: the `awk` range from `Id: <new-id>` to `===END_OF_MESSAGE===` contains `From:`, `To:`, `Title:`, `---`, and the end marker.
3. **AIF body is well-formed** (M2M only): the `<aif>...</aif>` block opens with a valid `@AIF/2.0` header.

On failure: fix own just-written message; if unable, append a `Protocol Violation - Self` notice to the log owner. Never edit another agent's message.

---

## What This Profile Does NOT Change in AIF Core

- Body schema (TASK, DELIVER, REVIEW_REQ, FEEDBACK, REVISE, QUERY, CLARIFY, ACK, CANCEL) — unchanged
- `<aif>...</aif>` wrapper and 3-layer fallback extraction — unchanged
- The rule that `TO: USER` triggers natural language output — unchanged
- `TRUNCATE`, `CONTENT_REF`, `CONTEXT_REF` semantics — unchanged; this profile adds a concrete backing store, not new rules
- `REF`, `SKILLS`, `MODEL`, `REPORT_TO` header fields — unchanged

---

## Relationship to `handoff_book_spec.md`

`handoff_book_spec.md` (pyCatchPokemon) is the **canonical reference implementation**. It further specifies:

- Role naming conventions (`Orchestrator`, `agent_pm_phase<N>`, `agent_rd_phase<N>`, …)
- Body length cap (≤ 2000 tokens; longer content goes in external files referenced by path)
- Read-access patterns (`grep`, `awk`, `tail -n N`) to avoid loading the full log
- Quick-checklist for writers (§9 of that spec)

Deployments using this profile inherit the envelope format and transport semantics defined here. Reference-implementation details (role naming, body length limit) are advisory.
