# Handoff: aif-dialect

> 這份文件給接手這個 repo 的開發者（或下一個 session）。

---

## 這個 Repo 是什麼

`monki103/aif-dialect` 是 AIF（Agent Interchange Format）2.0 的**獨立公開規格 repo**。
- 與模型無關（model-agnostic），不綁定 Claude / Copilot / Gemini
- 包含：規格（SKILL.md）、標註範例（examples/）、合規測試（tests/）、README（雙語）

---

## 目前狀態（2026-05-07）

### 已完成

| 項目 | 狀態 |
|------|------|
| SKILL.md — AIF/2.0 完整規格 | ✅ |
| SKILL.md — WHEN TO USE 使用時機判斷規則 | ✅ |
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

## 已知待辦 / 可能的下一步

### 短期
- [ ] **SKILL.md 版本標記**：目前標題寫 `AIF Dialect Skill`，但 agent-skills 那邊已升級至 v2.1（加了 signing、manifest 欄位）。若要同步 v2.1，需要決定是否要把 v2.1 功能合入這個公開 repo。
  - 目前這個 repo 維持 v2.0（較乾淨，沒有 Akasi-specific 擴充）
  - v2.1 的 `AUTH_METHOD`、`PUBKEY_REF`、`SIGNATURE`、`MANIFEST` 欄位屬於進階功能，可獨立成 v2.1 spec

- [ ] **合規測試執行器**：目前 `tests/` 是純文字 `.aif` 檔案，需手動或 LLM 驗證。可考慮加一個簡單的 Python runner 把測試案例送給模型判斷 valid/invalid。

- [ ] **更多實驗類型**：目前 README 的實驗都是 code generation 任務。可補充其他領域（writing、research、data analysis）的測試結果。

### 長期
- [ ] JSON-interop：定義 AIF ↔ JSON/OpenAI function call 的轉換標準
- [ ] 其他 CLI 整合：Gemini CLI、Copilot CLI 的安裝指引

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
**對應 agent-skills commit**：`fc3bda1`（main）  
**aif-dialect commit**：`3ff883c`（main）
