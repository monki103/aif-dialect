# Handoff: aif-dialect

> 這份文件給接手這個 repo 的開發者（或下一個 session）。

---

## 這個 Repo 是什麼

`monki103/aif-dialect` 是 AIF（Agent Interchange Format）2.0 的**獨立公開規格 repo**。
- 與模型無關（model-agnostic），不綁定 Claude / Copilot / Gemini
- 包含：規格（SKILL.md）、標註範例（examples/）、合規測試（tests/）、README（雙語）

---

## 目前狀態（2026-05-08）

### 已完成

| 項目 | 狀態 |
|------|------|
| SKILL.md — AIF/2.0 完整規格 | ✅ |
| SKILL.md — WHEN TO USE 使用時機判斷規則 | ✅ |
| SKILL.md — `<aif>` wrapper + 3-Layer Fallback Extraction | ✅ |
| SKILL.md — MUST 強制規則 + request→response pair | ✅ |
| SKILL.md — Skill Teaching Rule（零冗餘 spec 傳播） | ✅ |
| SKILL.md — 移除所有 Akasi 相關耦合 | ✅ |
| examples/ — 5 種訊息類型標註範例 | ✅ |
| tests/valid/ — 12 個合規測試案例 | ✅ |
| tests/invalid/ — 8 個違規測試案例 | ✅ |
| README.md — 英文 + 繁體中文雙語 | ✅ |
| 實驗結果整合進 README | ✅ |
| MIT License | ✅ |

### 實驗結果摘要（來自 agent-skills 實驗 harness）

| 實驗 | 模型 | AIF 品質 | NLU 品質 | 結果 |
|------|------|:--------:|:--------:|------|
| BlockOut PWA（手動）| Sonnet 4.6 | **92** | 75 | AIF +17 |
| BlockOut PWA | Sonnet 4.6 | **90** | 82 | AIF +8 |
| BlockOut PWA | Opus 4.7 | **83** | 79 | AIF +4 |

---

## SKILL.md 設計決策紀錄

### Skill Teaching Rule（2026-05-08）
收到含 `@AIF/2.0` header 的訊息，代表對方已具備 AIF 能力，不需要再附帶完整 spec。
委派給下游不熟悉 AIF 的 agent 時，用 `SKILLS: [aif-dialect]` header 委託 orchestrator 注入，而非自己內嵌 spec。

**設計邏輯：** Input token 一直不是瓶頸，Output NLU 廢話才是。但在多跳鏈（M1→M2→M3）中，最大的節省是 context window cascade——`TRUNCATE + CONTENT_REF` 截斷上游 payload，防止 context 爆炸。

### MUST 強制規則 + few-shot pair（2026-05-08）
加入明確的 MUST 規則（`<aif>` wrap，無 preamble）以及 request→response 範例 pair，對齊 few-shot enforcement 的最佳實踐。

### 移除 Akasi 耦合（2026-05-08）
移除 Akasi ↔ AIF M2M Terminology Bridge table 及所有 Akasi-specific 注解。
`aif-dialect` repo 維持純 model-agnostic 公開規格，Akasi-specific 擴充留在 `agent-skills`。

---

## AIF 定位說明（給推廣用）

**核心價值主張（不是「省 token」）：**
> 在 LLM Agent 之間建立語意契約，讓工作流可預測、可審計、可測試、可組合。Token 節省是副產品，不是目的。

**與競品的差異：**
| 項目 | AIF | MetaGPT Message | A2A / ACP |
|------|-----|-----------------|-----------|
| 層級 | Prompt 層 | Framework 層（Python） | 傳輸層 |
| 依賴 | 零（只需 system prompt） | Python 套件 | 基礎設施 |
| 內容規範 | ✅ 規定說什麼 | ❌ content 仍是自由文字 | ❌ 不規定內容 |
| 與 A2A 關係 | 互補（可疊加在 A2A 之上） | — | — |

**推廣切入點：**
- CrewAI Discussion #4111（task handoff schema 痛點）
- AutoGen Discussion #7144
- HN / Reddit（需先養 karma）

---

## 已知待辦 / 可能的下一步

### 短期
- [ ] **合規測試執行器**：目前 `tests/` 是純文字 `.aif` 檔案，需手動或 LLM 驗證。可考慮加一個簡單的 Python runner 把測試案例送給模型判斷 valid/invalid。

- [ ] **Benchmark 數據**：目前實驗都是 code generation 任務（BlockOut PWA）。需補充：
  - 複雜多輪流程（FEEDBACK/REVISE）的 token 對比數據（驗證 COMPACT + TRUNCATE 的效益）
  - 其他領域（writing、research、data analysis）

- [ ] **推廣貼文發布**：草稿已備妥，等待 GitHub 帳號發文。

### 中長期
- [ ] **v2.1 spec**：agent-skills 那邊已有 `AUTH_METHOD`、`PUBKEY_REF`、`SIGNATURE`、`MANIFEST` 欄位（signing + manifest）。決定是否合入公開 repo 或另立 v2.1 分支。
- [ ] **JSON-interop**：定義 AIF ↔ JSON / OpenAI function call 的轉換標準
- [ ] **其他 CLI 整合**：Gemini CLI、Copilot CLI 的安裝指引

---

## 關鍵檔案

```
aif-dialect/
├── SKILL.md          ← 主規格 + Claude Code skill 安裝點
├── README.md         ← 雙語說明 + 研究結果
├── examples/
│   ├── task.md       ← TASK + COMPACT 範例
│   ├── deliver.md    ← DELIVER COMPLETED / PARTIAL
│   ├── review_req.md ← ATTACH:REF + ATTACH:INLINE
│   ├── feedback.md   ← NEEDS_REVISION + APPROVED
│   └── query_clarify.md ← QUERY + CLARIFY pair
└── tests/
    ├── README.md     ← 合規測試使用說明
    ├── valid/        ← 12 個必須通過的訊息
    └── invalid/      ← 8 個必須被拒絕的訊息
```

---

## 與 agent-skills 的關係

- `agent-skills` repo（`yihsin428/agent-skills`）是實驗 harness + skill library 的主 repo
- `aif-dialect` 的 SKILL.md 源自 `agent-skills/skills/aif-dialect/SKILL.md`
- 兩者可能出現版本差異：agent-skills 跑得快、功能多；aif-dialect 是對外的穩定版

同步指令（從 agent-skills 更新到 aif-dialect）：
```bash
cp /path/to/agent-skills/skills/aif-dialect/SKILL.md /path/to/aif-dialect/SKILL.md
```

---

**建立日期**：2026-05-07  
**最後更新**：2026-05-08  
**對應 agent-skills commit**：`fc3bda1`（main）  
**aif-dialect 最新 commit**：`63ada25`（main）
