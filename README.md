# AIF Dialect — Agent Interchange Format 3.0

A **machine-to-machine (M2M) dialect** for hub-spoke agent
orchestration. A main-agent orchestrates sessionless sub-agents via a
file-backed message log; all inter-agent traffic is structured JSON,
never natural language.

## What is AIF v3?

AIF replaces natural-language exchange between agents with a compact,
structured envelope: a markdown header for routing and identity, a
single fenced JSON body for content, and an explicit terminator.

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
  "goal": "Build a Python CLI that watches a directory and POSTs file-change events to a webhook URL.",
  "deliverable": {
    "format": "milestone-list",
    "required_fields": ["id", "name", "depends_on", "est_effort_hours", "acceptance_criteria"]
  }
}
```
===END_OF_MESSAGE===
````

## Topology (hub-spoke)

```
USER ◀── NLU ──▶ MAIN-AGENT ◀── AIF ──▶ SUB-AGENT(s)
                     │
                     ▼
              handoff_book.md
```

- **main-agent** loads `SKILL.md`, owns `handoff_book.md`, orchestrates sub-agents
- **sub-agent** is sessionless, receives a ~300-token stub at spawn, never reads the file
- **handoff_book.md** is the single source of truth and audit log; main-agent is the sole writer

## Why v3 (vs v2.1)

v2.1 was a peer-to-peer M2M format that treated all agents as equals.
v3 acknowledges the **structural asymmetry** of real deployments: one
agent has the orchestration tools (file I/O, spawn API, NLU bridge to
the user) and the others don't. The spec now follows that asymmetry:

- `SKILL.md` is loaded **only by the main-agent** (~390 lines)
- Sub-agents receive a tiny inline stub at spawn time (~300 tokens)
- One message log, one writer, no race conditions
- 5 message TYPEs instead of 9 — application loops (review, retry,
  ack/cancel) encode as `TASK` with appropriate body
- Body is fenced JSON, eliminating NLU drift in agent-to-agent traffic

For the full design discussion and decision history, see
[`DESIGN.md`](DESIGN.md).

## Install (Claude Code)

The main-agent loads `SKILL.md` as a Claude Code skill:

```bash
mkdir -p ~/.claude/skills/aif-dialect
cp SKILL.md ~/.claude/skills/aif-dialect/SKILL.md
```

Sub-agents do **not** need this skill installed. At spawn time, the
main-agent inlines the stub from [`SKILL.md` §9](SKILL.md#9-sub-agent-stub)
into the sub's launch prompt.

## Message TYPEs (5)

| TYPE      | Direction       | Purpose                                  |
|-----------|-----------------|------------------------------------------|
| `TASK`    | main → sub      | Spawn a sub to do work                   |
| `DELIVER` | sub → main      | Sub returns result (incl. failure)       |
| `QUERY`   | sub → main      | Sub asks for clarification mid-task      |
| `CLARIFY` | main → sub      | Main answers a QUERY                     |
| `SUMMARY` | main → main     | Main checkpoints state                   |

v2.1's `REVIEW_REQ` / `FEEDBACK` / `REVISE` / `ACK` / `CANCEL` are
application-layer patterns in v3, encoded as `TASK` with appropriate
body shape.

## Repository structure

```
aif-dialect/
├── SKILL.md                  ← v3.0 normative spec (loaded by main-agent)
├── DESIGN.md                 ← Design rationale and decision history
├── examples/
│   ├── v3-walkthrough/       ← End-to-end + QUERY/CLARIFY walkthroughs
│   └── v2.1-legacy/          ← v2.1 example messages (historical)
└── tests/v2.1-legacy/        ← v2.1 conformance tests (historical)
```

Items marked **historical** are v2.1-era artifacts kept for reference.
The active normative spec is `SKILL.md` (v3.0).

---

## Research findings (v2.1 experiments)

The data below was collected against v2.1. The conclusions — structured
M2M outperforms NLU prose for agent-to-agent traffic — apply directly to
v3 and informed v3's stricter M2M policy.

**Setup**: Implementation model generates code → reviewer gives
FEEDBACK → implementer revises (max 1 round) → third-party evaluator
scores final output.

### Experiment results

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

### Key findings

**Quality**: AIF leads in all valid comparisons (margins: +17, +8, +4
points). Explicit acceptance criteria in the message body prevent the
model from interpreting requirements loosely.

**Token efficiency**: For complex tasks (BlockOut), structured M2M is
4–18% more efficient on output despite a slightly larger input prompt.
For simple tasks the comparison is noisy because revision-round
asymmetries dominate.

**Format echo**: Structured input (YAML/JSON) causes the model to mirror
the structure in its response. v3 leans into this by mandating fenced
JSON output via the §9 stub — the format echo becomes the protocol.

**Revision rounds**: When the pipeline is symmetric, structured and NLU
require the same number of review rounds. The quality difference comes
from first-pass correctness, not from fewer iterations.

---

# AIF Dialect — Agent Interchange Format 3.0（繁體中文）

一種用於 **machine-to-machine (M2M)** 的 hub-spoke agent 編排格式。
單一 main-agent 透過 file-backed message log 編排多個 sessionless
sub-agent；所有 agent 之間的溝通都是結構化 JSON，**不講人話**。

## 什麼是 AIF v3？

AIF 用結構化封包取代 agent 之間的自然語言交流：一段 markdown
header（負責 routing 與身分識別）、一段 fenced JSON body（負責內容）、
以及明確的終止符。

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
  "goal": "Build a Python CLI that watches a directory and POSTs file-change events to a webhook URL.",
  "deliverable": {
    "format": "milestone-list",
    "required_fields": ["id", "name", "depends_on", "est_effort_hours", "acceptance_criteria"]
  }
}
```
===END_OF_MESSAGE===
````

## 拓樸（hub-spoke）

```
USER ◀── NLU ──▶ MAIN-AGENT ◀── AIF ──▶ SUB-AGENT(s)
                     │
                     ▼
              handoff_book.md
```

- **main-agent**：載入 `SKILL.md`、擁有 `handoff_book.md` 寫入權、編排 sub-agent
- **sub-agent**：sessionless，spawn 時收到 ~300-token 的 stub，從不直接讀檔
- **handoff_book.md**：唯一的 ground truth 與 audit log；只有 main-agent 可寫

## 為什麼是 v3（vs v2.1）

v2.1 是 peer-to-peer 的 M2M 格式，把所有 agent 當成對等節點。v3 承認真實
部署中的**結構性不對稱**：某個 agent 一定有編排工具（檔案 I/O、spawn API、
NLU 對使用者的橋接），其他則沒有。Spec 反映這個事實：

- `SKILL.md` **只給 main-agent 載入**（~390 行）
- Sub-agent 在 spawn 時取得極簡 inline stub（~300 tokens）
- 單一 message log、單一 writer、無 race condition
- TYPE 從 9 種收斂為 5 種——application 層的迴圈（review / retry /
  ack / cancel）用 `TASK` + 適當 body 表達
- Body 強制 fenced JSON，徹底消除 agent 對 agent 的 NLU 漂移

完整設計討論與決策歷史見 [`DESIGN.md`](DESIGN.md)。

## 安裝（Claude Code）

由 main-agent 載入 `SKILL.md`：

```bash
mkdir -p ~/.claude/skills/aif-dialect
cp SKILL.md ~/.claude/skills/aif-dialect/SKILL.md
```

Sub-agent **不需要**安裝這個 skill。Spawn 時 main-agent 會把
[`SKILL.md` §9](SKILL.md#9-sub-agent-stub) 的 stub 內嵌到 sub 的
launch prompt。

## 訊息類型（5 種）

| TYPE      | 方向             | 用途                                       |
|-----------|------------------|--------------------------------------------|
| `TASK`    | main → sub       | spawn sub 執行工作                          |
| `DELIVER` | sub → main       | sub 回覆工作結果（含失敗）                  |
| `QUERY`   | sub → main       | sub 反問既有 TASK 的細節                    |
| `CLARIFY` | main → sub       | main 回應 QUERY                             |
| `SUMMARY` | main → main      | main 自我 checkpoint                        |

v2.1 的 `REVIEW_REQ` / `FEEDBACK` / `REVISE` / `ACK` / `CANCEL` 在 v3
屬於 application 層，用 `TASK` + 適當 body 表達。

## 專案結構

```
aif-dialect/
├── SKILL.md                  ← v3.0 規範性 spec（給 main-agent 載入）
├── DESIGN.md                 ← 設計理由與決策歷史
├── examples/
│   ├── v3-walkthrough/       ← 端到端 + QUERY/CLARIFY walkthrough
│   └── v2.1-legacy/          ← v2.1 範例訊息（歷史）
└── tests/v2.1-legacy/        ← v2.1 合規測試（歷史）
```

標示 **歷史** 的項目是 v2.1 時期的產物，保留供查考用。當前規範性 spec
是 `SKILL.md`（v3.0）。

## 研究數據（v2.1 實驗）

英文版上方的研究數據（quality / token efficiency / format echo /
revision rounds）原始實驗用 v2.1 進行，結論「結構化 M2M 優於 NLU」
直接適用於 v3，並推動了 v3 採用更嚴格的 M2M 政策（強制 JSON body、
拒絕 NLU 宣染）。
