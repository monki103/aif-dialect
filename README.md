# AIF Dialect — Agent Interchange Format 2.0

A structured message format for agent-to-agent (M2M) communication. Model-agnostic.

## What is AIF?

AIF replaces natural language in multi-agent pipelines with compact, structured key-value messages. It eliminates the "telephone game" problem — where information degrades as it passes through multiple agents — by making every field explicit and unambiguous.

```
@AIF/2.0
FROM: agent_pm
TO: agent_rd
TYPE: TASK
ID: T001
REF: -
REPORT_TO: agent_pm
---
GOAL: "Implement user authentication module"
ACCEPT:
  - A01: "Supports email/password login"
  - A02: "Returns JWT on success"
PRIORITY: HIGH
```

## When to Use AIF

**Use AIF** when communicating between agents (multi-step pipelines, review loops, delegation).

**Use natural language** when the receiver is a human user. Never send raw AIF to a user.

See [SKILL.md](SKILL.md) for full trigger conditions and format rules.

## Install (Claude Code)

```bash
cp SKILL.md ~/.claude/commands/aif-dialect.md
```

Then use `/aif-dialect` in any Claude Code session to load the spec.

## Message Types

| TYPE | Purpose | Reply |
|------|---------|-------|
| `TASK` | Assign a task | `DELIVER` |
| `DELIVER` | Report result | — |
| `REVIEW_REQ` | Request review | `FEEDBACK` |
| `FEEDBACK` | Review result | `REVISE` or — |
| `REVISE` | Resubmit after changes | `FEEDBACK` |
| `QUERY` | Ask a question | `CLARIFY` |
| `CLARIFY` | Answer a question | — |
| `ACK` | Confirm receipt | — |
| `CANCEL` | Cancel a task | `ACK` |

## Repository Structure

```
aif-dialect/
├── SKILL.md          ← Full spec + Claude Code skill loader
├── examples/         ← Annotated examples per message type
└── tests/
    ├── valid/        ← Conformance: messages that MUST parse correctly
    └── invalid/      ← Conformance: messages that MUST be rejected
```

## Conformance Testing

See [tests/README.md](tests/README.md) for how to use the test suite to verify an AIF-compatible implementation.

---

## Research Findings

The following experiments compare AIF against natural language (NLU) across real multi-agent pipelines. All experiments use a symmetric pipeline: same task, same reviewer model, same evaluation model — only the input format differs.

**Setup**: Implementation model generates code → reviewer gives FEEDBACK → implementer revises (max 1 round) → third-party evaluator scores final output.

### Experiment Results

| Experiment | Model | Task | Valid | AIF Quality | NLU Quality | Δ Input | Δ Output |
|---|---|---|:---:|---:|---:|---:|---:|
| BlockOut PWA (manual) | Sonnet 4.6 | 3D Tetris PWA | ✅ | **92** | 75 | +43.3%¹ | +21.8%¹ |
| MQL5 RateLimiter | Sonnet 4.6 | MQL5 class | ❌² | **92.5** | 91.3 | +33.2% | +20.5% |
| Smoke test | Sonnet 4.6 | Python add(a,b) | ❌³ | **99** | 99 | −75.1% | −94.7% |
| BlockOut PWA | Sonnet 4.6 | 3D Tetris PWA | ✅ | **90** | 82 | −4.1% | −18.4% |
| BlockOut PWA | Opus 4.7 | 3D Tetris PWA | ✅ | **83** | 79 | −3.8% | −1.6% |

¹ Estimated (chars/4), not measured from API.  
² VALID=false: payload asymmetry (AIF prompt was larger than NLU prompt due to harness config).  
³ VALID=false: NLU failed to produce code in R1 and needed an extra revision round, breaking round parity.

### Format Input Comparison (single-agent, no pipeline)

Sending the same code-review task in three formats to a single reviewer agent (Sonnet 4.6):

| Input Format | Input tokens | Output tokens | Δ Output |
|---|---:|---:|---:|
| NLU (plain English) | 6,610 | 327 | — |
| YAML | 6,621 | 392 | +19.9% |
| YAML+JSON | 6,615 | 543 | +66.1% |

Input token difference is negligible (~5–11 tokens) because the system prompt dominates. Output tokens increase with structured formats due to **format echo** — the model mirrors the input structure in its response.

### Key Findings

**Quality**: AIF leads in all valid comparisons (margins: +17, +8, +4 points). The `ACCEPT` field is the primary driver — explicit acceptance criteria prevent the model from interpreting requirements loosely.

**Token efficiency**: Counterintuitive for simple tasks. In the smoke test, AIF used 75% fewer input tokens because the NLU track failed R1 (produced no code), requiring an extra revision round that inflated NLU's total. In complex tasks (BlockOut), AIF is 4–18% more efficient on output despite a slightly larger input prompt.

**Format echo**: YAML and YAML+JSON input causes the model to respond in YAML format, adding AIF routing headers and multi-line `FIX:` blocks to every response. Pure AIF avoids this by using a distinct `@AIF/` marker and CAPS keys that signal "structured protocol" rather than "structured data."

**Revision rounds**: When the pipeline is symmetric, AIF and NLU require the same number of review rounds. The quality difference comes from first-pass correctness, not from needing fewer iterations.
