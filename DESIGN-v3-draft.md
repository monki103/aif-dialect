### Status

DRAFT — proposal only, not yet ratified.
Body schema, transport profile, and SKILL.md changes are NOT part of this
document. v2.1 spec under `SKILL.md` remains the active normative reference
until v3 is ratified.

Date: 2026-05-21
Branch: `claude/aif-v3-hub-spoke-design-VRDSK`

---

## 1. Goal

**M2M 不用講人話。**

Sub-agent ↔ main-agent 之間的訊息必須是結構化、可被 parse 的，不含 NLU
宣染（problem-restatement、polite preamble、emotive filler、explain-back
之類的東西）。NLU 只出現在 user ↔ main-agent 這一段。

這個目標比 v2.1 更嚴格。v2.1 允許 markdown body 自由書寫；v3 要求 body
本身就是結構化資料。

## 2. Topology (Hub-Spoke)

```
   USER  ◀── NLU ──▶  MAIN-AGENT  ◀── AIF ──▶  SUB-AGENT(s)
                          │
                          ▼
                   handoff_book.md     (file-backed; main owns)
```

### 角色職責

- **main-agent**
  - NLU ↔ AIF 翻譯
  - spawn / orchestrate sub-agents
  - 唯一對 `handoff_book.md` 有讀寫權的角色
  - 為每則訊息生成 `ID`
  - 把既有 message 從 handoff_book 抽出，inline 注入到新 sub-agent
    的 launch prompt 中（context assignment）
  - 必要時可摘要 handoff_book 後清空自身 context，再從摘要重新賦予
    自己 context（self-recovery）

- **sub-agent**
  - 接收 AIF；回覆 AIF
  - **sessionless** — 無自身狀態，不存歷史，可隨時被重新 spawn
  - 不直接讀寫 `handoff_book.md`
  - 不直接 spawn 下游 sub-agent

- **handoff_book.md**
  - append-only message log
  - ground truth
  - 同時支援：
    1. sub-agent 跨 spawn 的 context 連續性（透過 main 重新賦予）
    2. main-agent 自身的 context 壓縮與還原

### 設計後果

- Sub-agent 不需要載入完整 AIF spec；只需要極小的 reply schema（後續
  以 stub 形式定義）
- ID 的價值在於「sub-agent 可以在 reply 中引用既有訊息」，讓 main
  能拼湊出多輪 sub↔main 討論的線串
- v2.1 的 invariant **I3 (single-file installable)** 在 hub-spoke 模型下
  必須重新界定 — 詳見 §6 Pending

## 3. Header Schema (v3.0-draft)

每則 AIF 訊息以下列 header 起始：

```
# Title: <single-line subject>;
## Description: <optional one-paragraph context>;
*TO:* <role>@<model>;
*FROM:* <role>[@<model>];
*DATE:* YYYYMMDD-HHMMSS-NN;
*ID:* YYYYMMDD-HHMMSS-NN;
*IN-REPLY-TO:* <prior ID>;
*MESSAGE:*
<body>
```

### Field rules

| Field            | 必須 | 格式 / 規則                                                         |
|------------------|------|---------------------------------------------------------------------|
| `# Title`        | 是   | Markdown H1，單行；分號結尾                                         |
| `## Description` | 否   | Markdown H2；若出現則分號結尾；可為 "non" / 略                       |
| `*TO:*`          | 是   | `<role>@<model>`，e.g. `agent-pm@opus-4.7`；分號結尾                |
| `*FROM:*`        | 是   | `<role>` 或 `<role>@<model>`；`main-agent` 可省略 `@model`；分號結尾 |
| `*DATE:*`        | 是   | `YYYYMMDD-HHMMSS-NN`，與 `*ID:*` 內容相同（human-glance 用）        |
| `*ID:*`          | 是   | `YYYYMMDD-HHMMSS-NN`；由 main 在寫入 handoff_book 時生成            |
| `*IN-REPLY-TO:*` | 條件 | 引用先前訊息的 `ID`；sub→main 反問既有上下文時必填                  |
| `*MESSAGE:*`     | 是   | body 起點（**這一行不加分號**，因為其後接 body）                    |

### ID 規則細節

- 格式：`YYYYMMDD-HHMMSS-NN`
- `NN` = sub-second counter，00–99
- 規則：同一 `HHMMSS` 內 main spawn 多個訊息時遞增 `NN`，避免撞號
- 上限：每秒 100 則訊息。對 hub-spoke 拓樸足夠（main 不可能在 1 秒內
  spawn 100 個 sub）
- `DATE` 與 `ID` 內容相同 — 兩欄並存是為了讓人 glance log 時不必額外
  parse；機器 parse 任一即可

### 分號規則

- 所有 header 行強制以 `;` 結尾。
- 唯一例外：`*MESSAGE:*` 那行（後接 body，無分號）。
- 理由：明確的行分隔符有利於 model 在 fallback 抽取時定位欄位邊界。

## 4. Example

```
# Title: A new project analysis;
## Description: Initial scoping for the Q3 reporting refactor;
*TO:* agent-pm@opus-4.7;
*FROM:* main-agent;
*DATE:* 20260521-083312-00;
*ID:* 20260521-083312-00;
*MESSAGE:*
<body — structured, no NLU prose; schema TBD in §6>
```

Reply 範例（sub-agent → main，含上下文引用）：

```
# Title: Scoping plan for Q3 reporting refactor;
## Description: non;
*TO:* main-agent;
*FROM:* agent-pm@opus-4.7;
*DATE:* 20260521-083315-01;
*ID:* 20260521-083315-01;
*IN-REPLY-TO:* 20260521-083312-00;
*MESSAGE:*
<structured plan>
```

## 5. 為什麼這樣設計

| 決定                                  | 對照原始意圖                                       |
|---------------------------------------|----------------------------------------------------|
| Header 用 Markdown heading + `*KEY:*` | 接近 `handoff_book_spec.md` 的極簡格式             |
| 強制分號                              | 給 model 明確的欄位邊界（fallback 抽取友好）       |
| `TO: role@model`                      | main 需要從 header parse 出要呼叫哪個 model        |
| ID = DATE + NN                        | 單值即時間戳，無需額外 sequence file               |
| Sub 無檔案 I/O                        | 工具不對稱是結構性事實，spec 反映此事實            |
| Main 是唯一 writer                    | 單一 source of truth，避免 race；支援 self-recovery |

## 6. Pending（不在本 draft 內）

以下項目刻意留白，需後續討論／量測後再寫：

1. **Body schema** — 最小核心應只含 TASK / DELIVER 兩種，或更原始的
   「結構化 free-form」？「M2M 不用講人話」對 body 內部的約束程度需要
   明確化。
2. **Sub-agent QUERY 路徑** — header 已支援 `*IN-REPLY-TO:*`，但 body
   層級的 QUERY/CLARIFY schema 還沒定。Sub 反問是 v3 的一等公民（已
   確認），實作細節待定。
3. **I3「single-file installable」重新界定** — v2.1 的 I3 假設 spec
   要能 self-contained 載入；hub-spoke 模型下，main 載入完整 spec、sub
   只看 inline stub，I3 需降格為「Hub Spec 本身是 single-file」，不再
   是全域 invariant。
4. **Format-agnostic core** — 「連 JSON 都沒關係」是強原則還是
   aspirational？若強制，v3 Core 必須只定義語意，syntax 可替換
   （Markdown / JSON / YAML / KV 皆合法的同構表示）。
5. **handoff_book.md 角色** — 是 v3 的 default transport（規範性），
   還是 reference transport（示範性，可替換）？
6. **Token budget 量測** — 在另一個會跑實際 agent 的 repo 進行
   （aif-dialect 本身只是 spec repo，不適合做 runtime 量測）。
7. **跨平台適用性** — Copilot CLI / Gemini CLI / 純 LLM API 環境是否
   都符合 hub-spoke 的工具不對稱前提？若否，v3 是否需要 fallback 模式？

---

## Changelog (this draft)

- 2026-05-21: 初稿。確立 hub-spoke 拓樸、M2M strict 目標、header schema。
