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

---

---

# AIF Dialect — Agent Interchange Format 2.0（繁體中文）

一種用於 Agent 之間（M2M）溝通的結構化訊息格式。與模型無關。

## 什麼是 AIF？

AIF 在多 Agent pipeline 中以緊湊的結構化 key-value 訊息取代自然語言，消除「傳話遊戲」問題——資訊在多個 Agent 之間傳遞時逐漸失真——透過讓每個欄位都明確且無歧義來解決這個問題。

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

## 何時使用 AIF？

**使用 AIF**：當通訊對象是另一個 Agent 時（多步驟 pipeline、review loop、任務委派）。

**使用自然語言**：當接收方是人類使用者時。永遠不要直接把 AIF 訊息送給使用者。

完整觸發條件與格式規則請見 [SKILL.md](SKILL.md)。

## 安裝（Claude Code）

```bash
cp SKILL.md ~/.claude/commands/aif-dialect.md
```

在任何 Claude Code session 中執行 `/aif-dialect` 即可載入本規格。

## 訊息類型

| TYPE | 用途 | 預期回覆 |
|------|------|---------|
| `TASK` | 指派任務 | `DELIVER` |
| `DELIVER` | 回報結果 | — |
| `REVIEW_REQ` | 請求審查 | `FEEDBACK` |
| `FEEDBACK` | 審查結果 | `REVISE` 或 — |
| `REVISE` | 修改後重新提交 | `FEEDBACK` |
| `QUERY` | 提問 | `CLARIFY` |
| `CLARIFY` | 回答問題 | — |
| `ACK` | 確認收到 | — |
| `CANCEL` | 取消任務 | `ACK` |

## 專案結構

```
aif-dialect/
├── SKILL.md          ← 完整規格 + Claude Code skill 載入檔
├── examples/         ← 各訊息類型的標註範例
└── tests/
    ├── valid/        ← 合規測試：必須能正確解析的訊息
    └── invalid/      ← 合規測試：必須被拒絕的訊息
```

## 合規測試

使用測試套件驗證 AIF 相容實作的方式請見 [tests/README.md](tests/README.md)。

---

## 研究成果

以下實驗在真實多 Agent pipeline 中比較 AIF 與自然語言（NLU）。所有實驗採用對稱 pipeline：相同任務、相同 reviewer 模型、相同評估模型——只有輸入格式不同。

**實驗設定**：實作模型生成程式碼 → reviewer 給出 FEEDBACK → 實作模型修改（最多 1 輪）→ 第三方評估模型對最終輸出評分。

### 實驗結果

| 實驗 | 模型 | 任務 | 有效 | AIF 品質 | NLU 品質 | Δ 輸入 | Δ 輸出 |
|---|---|---|:---:|---:|---:|---:|---:|
| BlockOut PWA（手動） | Sonnet 4.6 | 3D 俄羅斯方塊 PWA | ✅ | **92** | 75 | +43.3%¹ | +21.8%¹ |
| MQL5 RateLimiter | Sonnet 4.6 | MQL5 class | ❌² | **92.5** | 91.3 | +33.2% | +20.5% |
| Smoke test | Sonnet 4.6 | Python add(a,b) | ❌³ | **99** | 99 | −75.1% | −94.7% |
| BlockOut PWA | Sonnet 4.6 | 3D 俄羅斯方塊 PWA | ✅ | **90** | 82 | −4.1% | −18.4% |
| BlockOut PWA | Opus 4.7 | 3D 俄羅斯方塊 PWA | ✅ | **83** | 79 | −3.8% | −1.6% |

¹ 估算值（字元數 ÷ 4），非 API 實測數據。  
² VALID=false：Payload 不對稱（因 harness 設定導致 AIF prompt 比 NLU prompt 大）。  
³ VALID=false：NLU 在 R1 未產出程式碼，需要額外一輪修改，破壞輪次對稱性。

### 輸入格式比較（單一 agent，無 pipeline）

以三種格式將相同的 code review 任務發送給單一 reviewer agent（Sonnet 4.6）：

| 輸入格式 | 輸入 tokens | 輸出 tokens | Δ 輸出 |
|---|---:|---:|---:|
| NLU（純英文） | 6,610 | 327 | — |
| YAML | 6,621 | 392 | +19.9% |
| YAML+JSON | 6,615 | 543 | +66.1% |

輸入 token 差異可忽略不計（~5–11 tokens），因為 system prompt 佔主導。輸出 tokens 隨結構化格式增加，原因是**格式回聲效應**——模型在回覆中鏡像輸入的格式結構。

### 主要發現

**品質**：AIF 在所有有效比較中均領先（差距：+17、+8、+4 分）。`ACCEPT` 欄位是主要驅動因素——明確的驗收條件防止模型對需求進行寬鬆詮釋。

**Token 效率**：簡單任務的結果違反直覺。在 smoke test 中，AIF 的輸入 tokens 少了 75%，原因是 NLU track 在 R1 失敗（未產出程式碼），需要額外一輪修改，拉高了 NLU 的總計。在複雜任務（BlockOut）中，AIF 的輸出效率高出 4–18%，儘管輸入 prompt 略大。

**格式回聲效應**：YAML 和 YAML+JSON 輸入會導致模型以 YAML 格式回覆，在每個回應中加入 AIF routing header 和多行 `FIX:` 區塊。純 AIF 透過使用獨特的 `@AIF/` 標記和大寫 key 來避免這個問題，這些設計傳達的是「結構化協議」而非「結構化資料」的語意。

**修改輪次**：在 pipeline 對稱的情況下，AIF 和 NLU 需要相同的審查輪次。品質差異來自首輪正確性，而非需要較少的迭代次數。
