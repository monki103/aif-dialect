# AIF Design Document

This document records the design rationale and invariants of AIF (Agent Interchange Format). It is **not a specification** — `SKILL.md` is the spec. This document explains *why* the spec is shaped the way it is, and what must remain true across future versions.

**Audience**:
- Spec maintainers deciding whether a proposed change fits AIF's design
- Profile authors deciding what belongs in core vs in a profile
- Application authors trying to understand where the AIF boundary is

---

## 1. What AIF Is

AIF is a **machine-to-machine (M2M) dialect** for agents sharing an LLM substrate. It specifies:

- How agent-to-agent messages are structured
- How messages are identified and cross-referenced
- How upstream context flows to a downstream agent

It does **not** specify:
- Who the agents are (role definitions belong to the application layer)
- When to switch between AIF and natural language (an application policy)
- How agents are dispatched, scheduled, or terminated
- How user-facing input is translated into M2M traffic

If a candidate rule talks about USER, PM, or routing, it does not belong in AIF.

---

## 2. Why AIF Exists

### 2.1 Three problems with naked NLU between agents

1. **Semantic ambiguity.** Agents trained to be helpful produce verbose, hedged replies. Downstream must re-parse intent on every hop; errors compound.
2. **Feedback bloat.** Reviewer→implementer cycles inflate token usage as each side wraps its message in NLU framing ("Thanks for sending this. I noticed a few things you might want to consider…").
3. **Sub-agent context loss.** Sub-agents are typically single-shot: an Agent-tool spawn gives the sub-agent its prompt, then discards the context when it returns. Without external persistence, cross-call memory is impossible.

### 2.2 What AIF does about each

| Problem | Mechanism |
|---|---|
| Ambiguity | Typed envelope + TYPE-specific body schema. The reader knows what fields exist before reading. |
| Feedback bloat | Structured fields (`VERDICT`, `ISSUES` with `SEV`/`LOCATION`/`DESC`/`FIX`) leave no place for NLU embellishment. |
| Sub-agent context loss | First-class **message ID** + a `RESOLVE(id)` contract backed by an interchangeable transport. |

The third item is the reason transport is core to AIF rather than an extension.

---

## 3. Design Invariants

These must hold across all spec versions. A change that violates one is by definition no longer AIF.

### I1. Role independence

AIF works regardless of who the agents are. Source examples may use suggestive names (`agent_a`, `agent_searcher`) but the spec must not require any specific role hierarchy, workflow shape, or agent vocabulary.

**Test**: remove all `roles/` files from this repo. The AIF spec must remain coherent.

### I2. Pure M2M scope

AIF describes how two agents talk to each other and how messages move between them. Everything else — when to engage AIF, how to translate to a user, who picks which model — belongs to the application layer.

**Test**: SKILL.md reads like a protocol spec, not a system architecture document.

### I3. Single-file installable

The minimal install of AIF is one file: `SKILL.md`. An agent given only this file as context must be able to send and receive valid AIF messages.

**Test**: an LLM with only SKILL.md as system context can produce a valid TASK and parse a valid DELIVER.

### I4. Semantic precision via schema

Every message has a typed envelope (`FROM`/`TO`/`TYPE`/`ID`/`REF`/`REPORT_TO`) and a TYPE-specific body schema. Unknown fields are ignored (forward compatibility); missing required fields cause rejection.

### I5. Structural compression of feedback

Cross-agent feedback uses structured fields, not prose. The schema actively forbids NLU framing by giving authors no field for it.

### I6. Transport is core, implementations are profiles

Persistent message addressability is a first-class AIF concern. The core spec defines:
- Message ID format requirements (uniqueness, addressability)
- A `RESOLVE(id) → message` contract — given an ID, downstream can obtain the referenced message
- That implementations may back this with a file, a broker, in-memory store, etc.

Concrete implementations live in `profiles/`. The abstraction lives in core.

**Test**: SKILL.md can describe the ID/RESOLVE contract without naming any specific transport.

### I7. Two context-handoff policies, one ID mechanism

Sub-agents are single-shot. When agent A dispatches to B, B starts with no prior context. AIF offers two policies for transferring upstream context:

| Policy | Field combo | Trade-off |
|---|---|---|
| **By-reference** | `CONTEXT_REF: [id1, id2, ...]` | B reads originals (high fidelity, more tokens) |
| **By-summary** | `CONTEXT_SUMMARY: "..."` + `CONTEXT_SOURCE: [id1, id2, ...]` | B reads A's digest (low tokens; source IDs preserved for audit) |

Both share one underlying mechanism: **every message has a globally addressable ID, and the transport provides `RESOLVE(id)`**. Without this, by-reference is impossible and by-summary cannot be audited.

The policies are non-exclusive: a TASK may carry both (summary + IDs) so B has a fast path and a verification path.

### I8. English-default for M2M traffic

The default working language for all M2M traffic is **English**. Agents SHOULD use English in:

- All AIF body free-form string fields (`GOAL`, `SUMMARY`, `DESC`, `NOTE`, etc.)
- Transport envelope fields that carry prose (e.g., `Title:` in file-backed transport)

**Override conditions**:
- The user-task explicitly requires another working language (e.g., a localization task, content authored in a target locale)
- The application has set a different M2M working language for the deployment

**Boundary rule**: translating the final result back to the user's natural language is the **application layer's** responsibility (the language gateway). AIF Core only stipulates that M2M defaults to English; the user-locale switch happens outside AIF.

**Rationale**: LLM substrates are uniformly fluent in English; defaulting reduces translation drift between agents, keeps logs consistent for audit, and matches the densest part of model training data. It also gives the application a clean rule to remember: *every hop inside AIF is English; the last hop out switches to user locale.*

**Test**: example M2M chains in SKILL.md and in profile examples are entirely English unless explicitly demonstrating the override case.

---

## 4. Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Application Layer  (role-aware, opinionated)                │
│  • Role definitions (agent_pm, agent_rd, agent_reviewer, …) │
│  • Routing policies (when NLU, when AIF, who dispatches)    │
│  • User-facing translation (language gateway)               │
│  • Commands / entry points                                  │
└─────────────────────────┬───────────────────────────────────┘
                          │ uses
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ AIF Core  (role-agnostic, pure M2M spec)                    │
│  • Message format (header + body, <aif> wrapper, fallback)  │
│  • TYPEs and body schemas                                   │
│  • ID system (uniqueness, REF / CONTEXT_REF / SOURCE)       │
│  • Transport abstraction (RESOLVE(id) contract)             │
│  • Two context-handoff policies (by-ref, by-summary)        │
└─────────────────────────┬───────────────────────────────────┘
                          │ implemented by
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Transport Profiles  (interchangeable implementations)       │
│  • file-backed-transport  (append-only log)                 │
│  • in-memory-transport    (single session, future)          │
│  • broker-transport       (Redis / SQS / NATS, future)      │
└─────────────────────────────────────────────────────────────┘
```

**Layering rule**: code in a lower layer must not name anything from a higher layer. AIF Core does not mention `agent_pm`. A transport profile does not interpret `TYPE: TASK` semantics — it treats the body as opaque payload.

---

## 5. What Belongs Where

### In AIF Core (`SKILL.md`)

- TYPE list and body schemas
- Header field definitions
- ID requirements (uniqueness, format constraints; concrete format deferred to transport)
- `RESOLVE(id)` contract: what callers can assume
- The two context-handoff policies (by-reference, by-summary)
- `<aif>` wrapper tag and fallback extraction rules
- Forward-compatibility rules (ignore unknown fields)
- **Default M2M working language: English (I8)** — the rule itself, not the override mechanism

### In Transport Profiles (`profiles/*.md`)

- Concrete ID format (e.g., `YYYYMMDD-HHMMSS-NN` for file-backed)
- Storage rules (append-only, immutability, etc.)
- `RESOLVE(id)` implementation (the actual lookup mechanism)
- Envelope additions (the transport may need its own headers — see file-backed `Date:`, `Title:`, `Lang:`)
- Self-verification rules

### In Application Layer (NOT in this repo's AIF spec)

- Role definitions (`roles/agent-*.md`)
- When to enter the AIF pipeline (auto-routing)
- MAIN vs SUB agent identification
- **User-locale translation at the boundary**: when the final hand-off goes to the user, the application's language gateway switches from English M2M to NLU in the user's locale (the I8 boundary rule, implemented application-side)
- Model-tier choice
- Override of I8 default (when the deployment chooses a non-English M2M working language)

---

## 6. Planned Changes for v2.1

Forward pointer for spec work (implemented under task (b) — not yet written):

- **Move out**: "When to Use AIF" section (currently in SKILL.md) → an application-side document. It describes when an application *chooses* AIF, which is not a protocol concern.
- **Move in**: Transport abstraction (currently profile-only) → a `Transport` section in SKILL.md, defining the `RESOLVE(id)` contract independent of any implementation.
- **Add**: `CONTEXT_SUMMARY` and `CONTEXT_SOURCE` body fields for the by-summary policy.
- **Add**: `READ_MODE: REQUIRED | OPTIONAL` qualifier on `CONTEXT_REF` lists (does the dispatcher require receiver to read all IDs, or is the list advisory?).
- **Generalize**: replace PM/RD/Reviewer agent names in spec examples with generic placeholders (`agent_a`, `agent_b`). Domain-suggestive names (`agent_searcher`) are fine — they're not the issue; the issue is hardcoding a specific workflow.
- **Add**: an explicit "Default Working Language" section in SKILL.md stating I8 (M2M = English by default; user-locale boundary is application's job).

### Undecided (open questions for v2.1)

- Does `MODEL` header (tier hint) belong in core or in an application-side header extension? It's about agent capability selection, which feels application-layer; but it travels in the protocol header.
- Does `SKILLS` header belong in core? It tells the receiver what to load — that's application policy, but it's serialized in the AIF message.

Default position: both stay in core for v2.1, flagged as "application-influenced fields". Revisit in v2.2 if a clean refactor emerges.

---

## 7. Explicitly Out of Scope

AIF does not address:

- User experience (UI, conversation pacing)
- Model selection beyond a coarse `MODEL` hint
- Authentication / signing (deferred to v2.2+; agent-skills already has draft ideas)
- JSON / function-call interop (a separate translation profile, not a core concern)
- Cost accounting (instrumentation, not protocol)
- Long-term agent identity / reputation (a higher-layer concern)

If a future contribution touches any of these, it is either a separate profile or it belongs in a different repo.

---

## 8. Relationship to Existing Files in This Repo

| File | Conformance with this design |
|---|---|
| `skills/aif-dialect/SKILL.md` | Mostly aligned. **Drift**: "When to Use AIF" section mixes in application policy (I2 violation). To be cleaned in v2.1. |
| `skills/aif-dialect/profiles/file-backed-transport.md` | Aligned. Implements I6 and I7. |
| `skills/universal-agent/SKILL.md` | **Application layer** — MAIN/SUB identification, reply-language rules. Not part of AIF; should not be loaded together with AIF as if it were a sub-component. |
| `skills/auto-routing/SKILL.md` | **Application layer** — names specific roles (`agent_pm` → `agent_rd` → …). Not part of AIF. |
| `skills/logging/SKILL.md` | **Application layer** — though it logs AIF messages, the policy of when/where/how to log is application's choice. |
| `roles/*.md` | Application layer. Currently most role files `SKILLS_REQUIRED: aif-dialect`; that direction is correct (role depends on protocol, not the reverse). PM v3.2 has gone further by making AIF self-activating; other roles have not caught up. |
| `commands/*.md` | Application layer. Already do not pre-load `aif-dialect` (self-activation). Aligned. |

Tasks (c) — cleaning up dependency direction in `universal-agent` and `auto-routing` — flows from this analysis: these files belong to the application layer and should describe AIF as an optional companion, not a required peer.

---

**Created**: 2026-05-21  
**Status**: Draft. The invariants (Section 3) are the ratchet — once accepted, future spec changes are tested against them.
