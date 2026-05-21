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
  必須重新界定 — 詳見 §6 Pending

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

## 4. Example

```
# Title: A new project analysis;
## Description: Initial scoping for the Q3 reporting refactor;
*TO:* agent-pm@opus-4.7;
*FROM:* main-agent;
*TYPE:* TASK;
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
*TYPE:* DELIVER;
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
| `TYPE:` enum                          | 讓 main 不靠語意推論就能 route / index 訊息        |
| ID = DATE + NN                        | 單值即時間戳，無需額外 sequence file               |
| 所有訊息必入 handoff_book              | 保證 `IN-REPLY-TO` 鏈不斷 + self-recovery 不缺片段 |
| 單值 `TO`，多收件人拆訊息              | 每個 sub 的 DELIVER 都能對齊唯一 spawn ID           |
| Sub 無檔案 I/O                        | 工具不對稱是結構性事實，spec 反映此事實            |
| Main 是唯一 writer                    | 單一 source of truth，避免 race；支援 self-recovery |

## 6. Pending（不在本 draft 內）

以下項目刻意留白，需後續討論／量測後再寫：

1. **Body schema (per TYPE)** — TASK / DELIVER / QUERY / SUMMARY 各自
   的 body 必填／選填欄位。已知約束：
   - TASK body 至少含 `GOAL:` 或 `INPUTS:`（純 forward 也算合法）
   - SUMMARY body 必含 `SOURCE_IDS:` 與摘要內容
   - DELIVER / QUERY body schema 待定
   - 走訪過的初版形狀見 `examples/v3-walkthrough/`，尚未 normative
2. **Message terminator** — handoff_book.md 是多訊息串接的單一檔，需
   要明確 message 邊界。Walkthrough 目前沿用 v2.1 的 `===END_OF_MESSAGE===`
   分隔符，但 v3 spec 還沒明文。候選：
   - 沿用 `===END_OF_MESSAGE===`（顯式，最穩）
   - 改用 `---` 或空行（簡潔但較依賴 parser 健壯度）
   - 不要分隔符，靠下一則 `# Title:` 作為 implicit boundary
3. **QUERY body schema 細節** — header 已支援 `*IN-REPLY-TO:*` 與
   `*TYPE: QUERY*`。Body 如何表達「我問的是什麼」（指定欄位 / 引用片段 /
   自由提問）尚未定。
4. **I3「single-file installable」重新界定** — v2.1 的 I3 假設 spec
   要能 self-contained 載入；hub-spoke 模型下 main 載入完整 Hub Spec、
   sub 只看 inline stub，I3 需降格為「Hub Spec 本身是 single-file」，
   不再是全域 invariant。
5. **Format-agnostic core** — 「連 JSON 都沒關係」是強原則還是
   aspirational？若強制，v3 Core 必須只定義語意，syntax 可替換
   （Markdown / JSON / YAML / KV 皆合法的同構表示）。
6. **Token budget 量測** — 在另一個會跑實際 agent 的 repo 進行
   （aif-dialect 本身只是 spec repo，不適合做 runtime 量測）。
7. **跨平台適用性** — Copilot CLI / Gemini CLI / 純 LLM API 環境是否
   都符合 hub-spoke 的工具不對稱前提？若否，v3 是否需要 fallback 模式？

### 已決議（不再 pending）

- `handoff_book.md` 是 v3 的**規範性 transport**（normative），非示範
  替代品；單一 writer = main-agent
- `*TYPE:*` 欄位入 Core header，初始 enum：TASK / DELIVER / QUERY / SUMMARY
- 多收件人 spawn 必拆成多則 TASK（單值 `TO`）
- 所有 AIF 訊息必入 handoff_book（無例外，含純 forward）
- main 視兩層遺忘（軟性 / 硬性）為 routine 機制，非 emergency

---

## Changelog (this draft)

- 2026-05-21: 初稿。確立 hub-spoke 拓樸、M2M strict 目標、header schema。
- 2026-05-21: 補 `*TYPE:*` 欄位（enum: TASK/DELIVER/QUERY/SUMMARY）、
  all-spawns-logged invariant、main 兩層 context 管理 + working state、
  單值 `TO` 規則。確認 `handoff_book.md` 為 normative transport。
- 2026-05-21: 加入端到端 walkthrough（`examples/v3-walkthrough/`）。
  Pending 新增「message terminator」項目（walkthrough 沿用
  `===END_OF_MESSAGE===` 待 spec 拍板）。
