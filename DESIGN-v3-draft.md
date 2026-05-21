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
  - **每次 sub-agent spawn 必產生一則 TASK 訊息進 handoff_book**
    （即使是純 forward，body 也以 `INPUTS:` 列出引用的 ID，確保
    `IN-REPLY-TO` 鏈不斷）
  - 把既有 message 從 handoff_book 抽出，inline 注入到新 sub-agent
    的 launch prompt 中（context assignment）
  - **兩層 context 管理**（v3 視為 routine，非 emergency）：
    - 軟性遺忘：每輪 sub 對話結束後，main 不再把 verbatim msg 重新
      塞進 prompt，改以 ID + working state 摘要引用
    - 硬性遺忘：在自然 checkpoint（cycle 結束 / token 接近上限）摘要
      handoff_book、重置自身 context，從摘要重新開機
  - 維護獨立的 **working state**（與 handoff_book 並列，結構不同）：
    ```
    current_goal: <user 原始請求的 1 行>
    current_step: <e.g. awaiting reviewer reply>
    recent_ids: [msg-id-7, msg-id-8]
    project_state: <3-5 行滾動摘要>
    open_questions: []
    ```

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
  必須重新界定 — 詳見 §7 Pending

## 3. Header Schema (v3.0-draft)

每則 AIF 訊息以下列 header 起始：

```
# Title: <single-line subject>;
## Description: <optional one-paragraph context>;
*TO:* <role>@<model>;
*FROM:* <role>[@<model>];
*TYPE:* TASK | DELIVER | QUERY | SUMMARY;
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
| `*TYPE:*`        | 是   | enum：`TASK` / `DELIVER` / `QUERY` / `SUMMARY`；分號結尾            |
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

### TYPE enum (initial set)

| TYPE      | 方向         | 用途                                                          |
|-----------|--------------|---------------------------------------------------------------|
| `TASK`    | main → sub   | 指派工作；包含純 forward 情境（body 至少含 `INPUTS:` 引用 IDs）|
| `DELIVER` | sub → main   | 回覆工作結果                                                  |
| `QUERY`   | sub → main   | 反問既有上下文；必填 `IN-REPLY-TO`（body schema 待定）         |
| `SUMMARY` | main → main  | main 產出的摘要訊息；body 必含 `SOURCE_IDS: [...]` 與摘要內容 |

Future types（`REVIEW_REQ` / `FEEDBACK` / `REVISE` / `ACK` / `CANCEL`
/ `CLARIFY`）是否進入 Core 待後續討論。預設它們屬於 application
extension 而非 Core。

### Recipient rule (single-value `TO`)

`*TO:*` 強制單值。一次要 dispatch 給 N 個 sub-agent **必須拆成 N 則
TASK，各有獨立 ID**。同樣 body 內容可重複；目的是讓每個 sub 的
DELIVER 都能用 `IN-REPLY-TO` 對齊到唯一的 spawn 訊息。

### Logging rule (invariant)

**所有 AIF 訊息（不論 TYPE）必須寫入 `handoff_book.md`，無例外。**

具體後果：

- 純 forward 的 spawn → 仍寫一則 `TYPE: TASK`，body 含 `INPUTS: [<id>, <id>]`
- main 的中間摘要 → 寫一則 `TYPE: SUMMARY`，body 含 `SOURCE_IDS: [...]`
  及摘要內容
- 這條規則是 main 的 hard invariant：「有 AIF message 必有 handoff_book
  entry」。違反會破壞 self-recovery 與 audit chain。

### Message terminator

每則 AIF 訊息以 `===END_OF_MESSAGE===` 單獨一行收尾。

- 該標記必須獨占一行，前後不得有 inline 內容
- 在 `handoff_book.md` 中，每則訊息嚴格遵守
  `header → body → 終止符 → 空行 → 下一則` 的順序
- 選擇此標記而非空行 / `---` / implicit boundary 的理由：
  - 顯式邊界對 model fallback 抽取最穩
  - 與 v2.1 / `handoff_book_spec.md` 維持連續
  - `---` 與 Markdown 表格 / horizontal rule / YAML front-matter 衝突

## 4. Body schema (per TYPE)

Body 接在 `*MESSAGE:*` 之後，至 `===END_OF_MESSAGE===` 為止。Body 以
**`KEY:` + value** 為主，列表用 YAML-style 縮排 bullet (`  - ...`)。
**禁止 NLU 宣染**（problem restatement / polite preamble / 自由段落）。

### 4.1 TASK body

| Field          | 必須 | 說明                                                                  |
|----------------|------|-----------------------------------------------------------------------|
| `GOAL:`        | 是   | 一行目標。即使是純 forward 也要寫，作為人類 glance 的索引              |
| `DELIVERABLE:` | 是   | 子欄位 `format` + `required_fields` + `acceptance`；告訴 sub 回什麼形狀 |
| `INPUTS:`      | 條件 | 引用的訊息 ID 列表。Fan-in / 純 forward 時必填                         |
| `SCOPE:`       | 否   | bullet list；scope 細節                                                |
| `CONSTRAINTS:` | 否   | bullet list；硬性限制                                                  |

### 4.2 DELIVER body

| Field      | 必須 | 說明                                                                  |
|------------|------|-----------------------------------------------------------------------|
| `STATUS:`  | 是   | enum：`COMPLETED` / `PARTIAL` / `FAILED`                              |
| `SUMMARY:` | 是   | 一行摘要                                                              |
| `RESULT:`  | 是   | 結構化 payload；shape 由觸發 TASK 的 `DELIVERABLE.format` 決定         |
| `NOTES:`   | 否   | 補充欄位 / 偏離說明（仍要結構化）                                      |

### 4.3 SUMMARY body

| Field         | 必須 | 說明                                              |
|---------------|------|---------------------------------------------------|
| `SOURCE_IDS:` | 是   | 被摘要訊息的 ID 列表                              |
| `SUMMARY:`    | 是   | 結構化摘要內容（key-value 為主，禁止自由段落）     |

### 4.4 QUERY / CLARIFY body

待 §7 確認後補（QUERY 屬 sub→main 反問，CLARIFY 為 main 對 QUERY 的
回覆）。CLARIFY 尚未加入 §3 的 TYPE enum，需一併拍板。

## 5. Example

```
# Title: A new project analysis;
## Description: Initial scoping for the Q3 reporting refactor;
*TO:* agent-pm@opus-4.7;
*FROM:* main-agent;
*TYPE:* TASK;
*DATE:* 20260521-083312-00;
*ID:* 20260521-083312-00;
*MESSAGE:*
<body — structured per §4, no NLU prose>
```

Reply 範例（sub-agent → main，含上下文引用）：

```
# Title: Scoping plan for Q3 reporting refactor;
## Description: non;
*TO:* main-agent;
*FROM:* agent-pm@opus-4.7;
*TYPE:* DELIVER;
*DATE:* 20260521-083315-01;
*ID:* 20260521-083315-01;
*IN-REPLY-TO:* 20260521-083312-00;
*MESSAGE:*
<structured plan>
```

## 6. 為什麼這樣設計

| 決定                                  | 對照原始意圖                                       |
|---------------------------------------|----------------------------------------------------|
| Header 用 Markdown heading + `*KEY:*` | 接近 `handoff_book_spec.md` 的極簡格式             |
| 強制分號                              | 給 model 明確的欄位邊界（fallback 抽取友好）       |
| `TO: role@model`                      | main 需要從 header parse 出要呼叫哪個 model        |
| `TYPE:` enum                          | 讓 main 不靠語意推論就能 route / index 訊息        |
| ID = DATE + NN                        | 單值即時間戳，無需額外 sequence file               |
| 所有訊息必入 handoff_book              | 保證 `IN-REPLY-TO` 鏈不斷 + self-recovery 不缺片段 |
| 單值 `TO`，多收件人拆訊息              | 每個 sub 的 DELIVER 都能對齊唯一 spawn ID           |
| Sub 無檔案 I/O                        | 工具不對稱是結構性事實，spec 反映此事實            |
| Main 是唯一 writer                    | 單一 source of truth，避免 race；支援 self-recovery |

## 7. Pending（不在本 draft 內）

以下項目刻意留白，需後續討論／量測後再寫：

1. **QUERY / CLARIFY body schema + TYPE enum 擴充** — QUERY (sub→main
   反問) 與 CLARIFY (main→sub 回應) 一組。需決定：
   - CLARIFY 是否加入 §3 TYPE enum（變 5-entry）
   - QUERY body 表達「問什麼」的欄位設計（FIELDS / QUESTION / BLOCKING 等）
   - CLARIFY body 回覆欄位
   - QUERY 是否容忍 sub 自行決策（非 blocking）vs 強制阻塞等待
   走訪場景由 step C 提供。
2. **I3「single-file installable」重新界定** — v2.1 的 I3 假設 spec
   要能 self-contained 載入；hub-spoke 模型下 main 載入完整 Hub Spec、
   sub 只看 inline stub，I3 需降格為「Hub Spec 本身是 single-file」，
   不再是全域 invariant。
3. **Format-agnostic core** — 「連 JSON 都沒關係」是強原則還是
   aspirational？若強制，v3 Core 必須只定義語意，syntax 可替換
   （Markdown / JSON / YAML / KV 皆合法的同構表示）。
4. **Token budget 量測** — 在另一個會跑實際 agent 的 repo 進行
   （aif-dialect 本身只是 spec repo，不適合做 runtime 量測）。
5. **跨平台適用性** — Copilot CLI / Gemini CLI / 純 LLM API 環境是否
   都符合 hub-spoke 的工具不對稱前提？若否，v3 是否需要 fallback 模式？

### 已決議（不再 pending）

- `handoff_book.md` 是 v3 的**規範性 transport**（normative），非示範
  替代品；單一 writer = main-agent
- `*TYPE:*` 欄位入 Core header，初始 enum：TASK / DELIVER / QUERY / SUMMARY
- 多收件人 spawn 必拆成多則 TASK（單值 `TO`）
- 所有 AIF 訊息必入 handoff_book（無例外，含純 forward）
- main 視兩層遺忘（軟性 / 硬性）為 routine 機制，非 emergency
- Message terminator = `===END_OF_MESSAGE===`（單獨一行；前後留空行）
- TASK / DELIVER / SUMMARY body schema 為 normative（§4），key:value
  + YAML 縮排 bullet、禁 NLU 宣染

---

## Changelog (this draft)

- 2026-05-21: 初稿。確立 hub-spoke 拓樸、M2M strict 目標、header schema。
- 2026-05-21: 補 `*TYPE:*` 欄位（enum: TASK/DELIVER/QUERY/SUMMARY）、
  all-spawns-logged invariant、main 兩層 context 管理 + working state、
  單值 `TO` 規則。確認 `handoff_book.md` 為 normative transport。
- 2026-05-21: 加入端到端 walkthrough（`examples/v3-walkthrough/`）。
  Pending 新增「message terminator」項目（walkthrough 沿用
  `===END_OF_MESSAGE===` 待 spec 拍板）。
- 2026-05-21: 鎖定 message terminator = `===END_OF_MESSAGE===`（單獨一行）。
  §3 補上 "Message terminator" 小節；§6 移除該 pending 項並列入已決議。
- 2026-05-21: 加入 §4 Body schema (per TYPE)，TASK / DELIVER / SUMMARY
  body 升格 normative；QUERY / CLARIFY 整併為 pending item 1（含 TYPE
  enum 是否擴充至 5 entry）。其餘節次相應後移為 §5/§6/§7。
