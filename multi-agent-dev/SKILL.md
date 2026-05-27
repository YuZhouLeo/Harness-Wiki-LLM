---
name: multi-agent-dev
description: 啟動多代理開發協作工作流（planner → Codex advisor → executor → Codex impl-validator → lesson-keeper），用於把非平凡的需求拆解、規劃、實作、對抗式驗證、萃取經驗。**僅在使用者明確說「啟動多代理開發」、「跑多代理流程」、「multi-agent dev mode」、`/multi-agent-dev` 顯式叫起，或請求「用多代理流程做 X」、「啟動多代理工作流」時觸發**。DO NOT auto-trigger on：一般 coding/debug/refactor 問答、單檔修改、純研究或閒聊、想要快速答案、無時間跑完整輪次的場景。觸發後主代理只調度，不寫 code、不跑 Bash、不做技術判斷——全交給子代理與 Codex。
---

# 多代理工作流：主代理調度手冊

> 觸發語：「**啟動多代理開發**」（或語意等價：「跑多代理流程」、「用 multi-agent workflow 做 X」、「multi-agent dev mode」）；或顯式 `/multi-agent-dev`。
>
> 對應規範：[多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md)（特別是 §2、§3、§4、§5）。

---

## 路徑慣例（重要）

| 類別 | 範例 | 路徑寫法 |
|---|---|---|
| 工作流自身資產 | spec / templates / codex-prompts / 工作流 ADR (0001-0006) / 全域 settings | **絕對路徑** `C:/Users/yuzho/.claude/skills/multi-agent-dev/...` 或 `C:/Users/yuzho/.claude/settings.json` |
| 各專案產出 | `wiki/runs/<時間戳>-<功能>.md`、`wiki/runs/INDEX.md`、`wiki/lessons.md`、`plans/<功能>計劃.md`（**使用者寫**）、`plans/<功能>-工作拆分.md`（**planner 寫**）、`plans/<功能>-handoff-*.md`、`plans/<功能>計劃-advisor-stuck-*.md` | **cwd 相對路徑** |
| 各專案資產 | `AGENTS.md`、`docs/invariants.md`、`docs/conventions/`、**專案業務 ADR `docs/ADR/0007+`** | **cwd 相對路徑** |

> 反例（**禁止**）：`C:/Users/yuzho/.claude/skills/multi-agent-dev/wiki/lessons.md` 這種把全域路徑加上「各專案產出」後綴的寫法——會把所有專案的 lessons 都寫進全域，**絕對錯誤**，應改回 `wiki/lessons.md`。

---

## 前置條件：SendMessage 工具必須可用

本流程在步驟 3、5、8 等多處使用 `SendMessage(to=<agent_id>, ...)` 把上下文續送給**同一個** subagent，避免重新 spawn 損失歷史。此工具受實驗 flag 控制：

- 必須條件：[C:/Users/yuzho/.claude/settings.json](C:/Users/yuzho/.claude/settings.json) 頂層含 `"env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" }`
- 啟動驗證：在主代理內執行 `ToolSearch select:SendMessage`，能取回 schema 即可用；回 `No matching deferred tools found` 表示未啟用，**立刻停手**並請使用者修正 settings.json
- 詳細測試與排錯：見 [SendMessage子代理續用使用指南.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/SendMessage子代理續用使用指南.md)

如未啟用就硬跑流程，會在步驟 3 第一次 SendMessage 時失敗，整個 run 報廢。

---

## Codex plugin 介面適配（1.0.4）

> 這份 handbook 的 codex 呼叫已適配 codex-plugin-cc **1.0.4** 的實際 CLI surface。原規範（spec/多代理協作工作流計劃.md + ADR-0002/0003）假設了 plugin 接受「檔案路徑 + 自訂 prompt」與 `--resume <session-id>`，但實機 1.0.4 沒有這些介面。下列三條是當前真正可用的呼叫慣例，**全部步驟以本節為準**，與 spec / ADR 衝突時以本節覆蓋：

1. **檔案餵法**：plugin **只看 git working tree diff**，不收檔案路徑參數。
   - planner 寫完計劃書後，主代理 `git add <plans/X計劃.md>`（staged 不 commit），Codex 就能在 working tree 看到該檔
   - executor 寫完實作後檔案本來就在 working tree（無論 staged 與否，plugin 會看到）
   - 換言之：**Codex 看到什麼 = git working tree 此刻的狀態**

2. **Prompt 注入法**：plugin 沒有「自訂完整 prompt」，但 `/codex:adversarial-review --background "[focus] <text>"` 的 focus text 會被 plugin 加進 review prompt。
   - 把 codex-prompts/ 下對應的 prompt 全文當 focus text 塞進去
   - **兩個階段都用 `/codex:adversarial-review`**（plugin 1.0.4 中 `/codex:review` 不收 focus，只有 adversarial-review 收）；用 focus text 把行為從「對抗 reviewer」拉成「諮詢 advisor」或「對抗 impl-validator」

3. **Session 串接法**：plugin 只有 `--resume-last`，沒有 `--resume <id>`。
   - 序列保證：advisor 4 輪全跑完才開始 impl-validator——這樣 `--resume-last` 在每個 phase 內都指對該 phase 的 session
   - **絕不交錯**：advisor 中途插一個 impl review、或 impl 中途回頭問 advisor，會讓 `--resume-last` 接到錯的 session 直接污染兩條歷史。觸發終局條件（advisor-stuck / handoff）時立刻停手，禁止跨 phase 補問
   - session-id 仍從 `--background` 回傳擷取記入 Run Log（用於 audit / 對話追溯，不用於 `--resume`）

下方步驟 2/4/7/8 的指令格式已照本節寫，不必再轉譯。

---

## 進入多代理模式時，你（主代理）的角色

**只調度，不生產。**判斷分兩類：**結構性判斷可以做、內容性判斷一律禁止**。

### 可做：結構性判斷（語法 / 字面層次，不涉及對錯）

- PASS / FAIL 字串是否出現在 Codex 回應結尾
- 輪數計數、是否達上限（含結構性 3 輪 / 非結構性 2 輪的區分，見步驟 5）
- 從 Codex 建議擷取 `[structural]` / `[non-structural]` 標籤
- 子代理回傳是否含預期的檔案路徑、`agentId`、`session_id` 字串
- 是否觸發終局條件（次數達標、字串命中）
- 整理對答歷史成 handoff / advisor-stuck MD（**搬字而非評論**）

### 不准做：內容性判斷（對錯 / 好壞 / 該不該）

- 架構設計是否合理（→ advisor）
- 程式碼是否正確（→ impl-validator）
- 計劃書是否完整（→ advisor）
- 建議是否有道理（→ planner 回應）
- advisor 與 planner 的立場誰對（→ 使用者裁決）
- 「這個應該改成 X」「這個套件選對沒」（→ Codex / 子代理）

### 不准做：直接動手

- 不准 Read 程式碼（除終局 MD 整理對答歷史外）
- 不准跑 Bash（git / npm / test 等；**例外**：步驟 2 / 步驟 4 的 `git add <plans/檔>` 屬於 Codex 餵檔機制，不算技術判斷）
- 不准修改任何檔案，**除了**：
  - **Run Log**（`wiki/runs/<時間戳>-<功能 slug>.md`）：每次子代理回應 / Codex 回傳時同步追加一行
  - **Handoff / Advisor-stuck MD**：觸發 §3.2.1 / §3.2.2 條件時（搬字而非評論）

---

## 工作流程（線性 11 步）

### 步驟 0：建立 Run Log

工作流啟動的第一步——

1. 取得當前時間：`YYYYMMDD-HHmm`
2. 從使用者需求抽出功能名 slug（盡量短；含中文 OK）
3. **Write** `wiki/runs/<時間戳>-<功能 slug>.md`（**cwd 相對**；若 `wiki/runs/` 不存在請先 mkdir，並建 `wiki/runs/INDEX.md` 骨架），內容初始化為：

   ```markdown
   # Run: <功能名> — <YYYY-MM-DD HH:mm>

   ## Agent ID 索引（SendMessage 用）

   | 角色 | agent_id | 說明 |
   |---|---|---|
   | planner | _待填_ | 規劃代理；步驟 1 spawn |
   | executor | _待填_ | 執行代理；步驟 6 spawn |
   | lesson-keeper | _待填_ | 經驗萃取代理；步驟 9 spawn（PASS 時） |

   ## Codex Session ID 索引（audit 用；後續 round 改用 `--resume-last`）

   | 角色 | session_id | 說明 |
   |---|---|---|
   | advisor | _待填_ | 計劃顧問；步驟 2 開（`/codex:adversarial-review --background "[focus] <advisor-prompt 全文>"`） |
   | impl-validator | _待填_ | 實作驗證者；步驟 7 開（`/codex:adversarial-review --background "[focus] <impl-validator-prompt 全文>"`） |

   > 1.0.4 介面沒有 `--resume <id>`，session id 只供事後追溯；後續輪次靠序列保證 + `--resume-last` 接續。

   ## 時間線

   [HH:mm:ss] [orchestrator] Workflow start — feature="<功能名>"
   ```

4. 後續每收到子代理 / Codex 回應 → **同步**追加一行（不要批次留到最後）；同時把新拿到的 `agent_id` / `session_id` 填到上方索引表，方便後續 SendMessage / `--resume` 引用

### 步驟 1：規劃 → 工作拆分

> **改動（v0.3）**：使用者在啟動工作流前已自己寫好計劃書 `plans/<功能名>計劃.md`（範本：[templates/user-plan-template.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/templates/user-plan-template.md)），含 §1 功能目標 + §2 acceptance criteria 等。planner 的職責**不再是寫計劃書**，而是讀使用者計劃書後 → 輸出**工作項拆分**到 `plans/<功能名>-工作拆分.md`，含：
>
> - 工作項 Task A / Task B / Task C / … （每個 ≈ 一個 commit 量）
> - 每項標 `[depends-on: Task-X]` 標相依關係、`[parallel]` 標可並行
> - planner 自行判斷可拆分執行（並行）或必須順序執行（合併）
> - acceptance criteria → Task 對應表（確認每條 §2 條件都被覆蓋）
> - 假設清單（使用者計劃書缺欄位時 planner 推斷的部分）

- 確認 `plans/<功能名>計劃.md` 已存在（**使用者寫好的**，cwd 相對）；若不存在 → 回報使用者「先依範本寫好計劃書再啟動」並停手
- 呼叫 `Agent(subagent_type="planner", "讀 plans/<功能名>計劃.md → 依範本 templates/user-plan-template.md 規格判讀 → 輸出工作拆分到 plans/<功能名>-工作拆分.md")`
- 從 Agent 回傳結尾擷取 `agentId: <hash>`，記為 `<planner_id>`
- **立即更新** Run Log 上方「Agent ID 索引」表的 planner 欄位
- 收工作拆分檔路徑 + 重點摘要（Task 數量、可並行 Task 數）
- Run Log 時間線追加：`[HH:mm:ss] [planner] Spawn — id=<planner_id>`、`[HH:mm:ss] [planner] Return — plans/<功能>-工作拆分.md (<Task 數> tasks, <parallel 數> parallel)`

### 步驟 2：計劃顧問協商（諮詢模式 Codex）

> 介面適配：用 `/codex:adversarial-review`（不是 `/codex:review`，後者 1.0.4 不收 focus），靠 focus text 把行為從「對抗 reviewer」拉成「諮詢 advisor」。Codex 看 working tree，所以要先把兩份計劃檔 staged。
>
> **prompt 注入簡化（v0.3）**：focus text 只塞**精簡核心**（角色 + 四大面向 + 嚴重度標籤 + 輸出格式 + 必讀清單路徑），不再貼全文——advisor-prompt.md 的細節由 Codex 自己 read 該檔絕對路徑取得，省 token。

- 第一步：Bash `git add plans/<功能>計劃.md plans/<功能>-工作拆分.md`（staged 不 commit，讓 Codex 在 working tree 看到兩檔）
- 第一輪指令（單行）：
  ```
  /codex:adversarial-review --background "[focus] 諮詢模式（advisor）。對 staged 的 plans/<功能>計劃.md + plans/<功能>-工作拆分.md 提建議——不是 PASS/FAIL 守門，是諮詢式提案。

  完整指引請先 read C:/Users/yuzho/.claude/skills/multi-agent-dev/codex-prompts/advisor-prompt.md 全文後再執行；審查基線見 C:/Users/yuzho/.claude/skills/karpathy-guidelines/SKILL.md。

  必做：每條建議標 [structural] 或 [non-structural] 標籤，主代理會依此分配協商輪數上限。輸出格式：逐條編號 + [structural|non-structural] [工作拆分|架構|語法|工具] 標籤 + 建議 + 理由。最後一行只寫「<N> 條建議（結構性 X / 非結構性 Y）」"
  ```
- 收建議清單 + **advisor-session-id**（記下來，供 audit；後續輪次不用 `--resume <id>`，用 `--resume-last`）
- Run Log 追加：`[HH:mm:ss] [plan-advisor] adversarial-review --background (advisor focus) — session=<advisor-session-id>`、`[HH:mm:ss] [plan-advisor] Return — N suggestions (structural=X, non-structural=Y)`

### 步驟 3：轉交規劃代理

- `SendMessage(to="<planner_id>", summary="relay advisor suggestions", message="顧問建議如下，逐條回應採納/不採納+理由：<貼建議清單>")`
- 此呼叫**非阻塞**，主代理會在背景收到 task-notification 後拿到結果（含 planner 的採納記錄與同一個 `<planner_id>`，可繼續下一輪 SendMessage）
- **絕對禁止** 第二輪重 `Agent(subagent_type="planner", ...)`——會失去前輪對答歷史，違反 [SendMessage 使用指南](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/SendMessage子代理續用使用指南.md) §核心原則
- Run Log 追加 `[planner] SendMessage — to=<planner_id>` + `[planner] Return — accepted X / rejected Y`

### 步驟 4：轉回顧問

> 介面適配：1.0.4 沒有 `--resume <id>`，用 `--resume-last`。**因為 step 7 前都還沒開過 impl-validator session，這個 `--resume-last` 必然指到 advisor session**——只要沒在 advisor 階段中途穿插任何其他 `/codex:adversarial-review` 呼叫，序列就是安全的。

- 若 planner 改寫過計劃書或工作拆分：先 Bash `git add plans/<功能>計劃.md plans/<功能>-工作拆分.md` 把新版重新 stage（確保 Codex 看到新內容）
- 指令：`/codex:adversarial-review --resume-last "[focus] 諮詢 round N。規劃代理回應如下，請判斷哪些建議達成共識、哪些仍要再議：<貼 planner 回應>"`
- 收顧問回應（接受 / 再反駁 / 新建議；新建議仍要標 `[structural]` / `[non-structural]`）
- Run Log 追加

### 步驟 5：協商迴圈（結構性 3 輪 / 非結構性 2 輪）

> **改動（v0.3）**：協商輪數依建議嚴重度分級——`[non-structural]` 2 輪上限、`[structural]` 3 輪上限。主代理依 advisor 標的標籤計數，不做內容性判斷。

- 重複步驟 3–4，直到全部建議達成共識或觸發下方輪數規則
- 中途**採納**的建議要 SendMessage 請規劃改寫計劃書 / 工作拆分
- **未採納但顧問接受**的要請規劃在工作拆分檔的「假設」或「架構決策」段補記「曾考慮 X，不採用因為 Y」

#### 輪數計算規則（主代理執行）

每條建議獨立計輪數：

| 標籤 | 上限 | 達上限後 |
|---|---|---|
| `[non-structural]` | 2 輪 | **軟性終局**：主代理自動視為「顧問接受規劃方案」，請 planner 補記但不寫 advisor-stuck MD，繼續往下走 |
| `[structural]` | 3 輪 | **硬性終局**：寫 advisor-stuck MD 交棒（見下方終局條件 A） |

混合情境：若某輪同時包含 `[structural]` 與 `[non-structural]` 建議，分開計輪：`[non-structural]` 第 2 輪達上限就自動收，`[structural]` 繼續到第 3 輪。

#### 終局條件 A（advisor-stuck，僅 `[structural]` 觸發）

3 輪完整來回協商後 `[structural]` 仍互不退讓 → 寫 `plans/<功能名>計劃-advisor-stuck-<YYYYMMDD-HHmm>.md`（**cwd 相對**；複製 [advisor-stuck-template.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/templates/advisor-stuck-template.md) 骨架，**先過 template 內 lint checklist**，填入完整對答歷史與 `[structural]` 標籤）→ Run Log 追加 `[orchestrator] Termination triggered — advisor 3-round no-consensus on structural items` → **立刻停手等使用者裁決**。INDEX.md 追加一行（結果欄寫 `⚠️ advisor-stuck`）。

### 步驟 6：執行

- 共識達成後，呼叫 `Agent(subagent_type="executor", "依 plans/<功能>計劃.md（使用者計劃書）+ plans/<功能>-工作拆分.md（planner 工作項）實作。請依工作拆分檔的 Task 順序與相依關係執行；可並行 Task 可同步進行")`
- 從回傳擷取 `agentId` 並記為 `<executor_id>`，**立即更新** Run Log「Agent ID 索引」表 executor 欄位
- 收檔案清單 + lint/test 輸出 + Task 完成狀態
- Run Log 追加：`[HH:mm:ss] [executor] Spawn — id=<executor_id>`、`[HH:mm:ss] [executor] Return — <檔案清單>, lint=..., test=..., tasks=N/N`

### 步驟 7：實作驗證（對抗模式 Codex）

> 介面適配：用 `--background` 開**新** session（不要 `--resume-last`，否則會接到 advisor session 污染對抗模式）。executor 改的檔本來就在 working tree，不需要額外 `git add`。
>
> **prompt 注入簡化（v0.3）**：focus text 只塞精簡核心（角色 + 5 條檢查項 + 嚴重度 + 輸出格式），不再貼全文——impl-validator-prompt.md 的細節由 Codex 自己 read。

- 第一輪指令（單行）：
  ```
  /codex:adversarial-review --background "[focus] 對抗模式（impl-validator）。對 working tree 內 executor 剛實作的檔做 PASS/FAIL 守門。

  完整指引請先 read C:/Users/yuzho/.claude/skills/multi-agent-dev/codex-prompts/impl-validator-prompt.md 全文後再執行；審查基線見 C:/Users/yuzho/.claude/skills/karpathy-guidelines/SKILL.md。

  計劃書：plans/<功能>計劃.md；工作拆分：plans/<功能>-工作拆分.md（兩檔已 staged）
  executor 回的 lint/test 結果：<貼 lint/test 輸出>

  輸出格式：每條 FAIL 配 `<file>:<line>` + 嚴重度 (critical/major/minor)。最後一行只寫 PASS 或 FAIL"
  ```
- 收結果 + **impl-session-id**（記下來供 audit）
- Run Log 追加：`[HH:mm:ss] [impl-validator] adversarial-review --background — session=<impl-session-id>`、`[HH:mm:ss] [impl-validator] Return — PASS/FAIL (N findings)`

### 步驟 8：FAIL 迴圈

- **若 FAIL**：`SendMessage(to="<executor_id>", summary="impl FAIL round N", message="Codex 駁回理由如下，依此修正：<貼 FAIL 理由原文>")`
  - 同一個 executor 帶完整歷史（先前讀過的檔案、實作 reasoning）續跑，不需要重塞 plan + 既有實作脈絡
  - **絕對禁止** 第二輪重 `Agent(subagent_type="executor", ...)`——同上理由
  - 拿到回傳後 Run Log 追加 `[executor] SendMessage — round=N, to=<executor_id>` + `[executor] Return — <新版檔案清單>`
- 重審：`/codex:adversarial-review --resume-last "[focus] impl round N。executor 已依上輪 FAIL 理由修正 working tree 內檔案，請重審 PASS/FAIL"`
  - 因為 step 7 已開了 impl session，且 advisor session 已結束、不會再呼叫，`--resume-last` 必然指到 impl session
- 回到步驟 7 結果判讀

#### 終局條件 B（handoff）

連續 3 輪 FAIL → 寫 `plans/<功能名>-handoff-<YYYYMMDD-HHmm>.md`（**cwd 相對**；複製 [handoff-template.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/templates/handoff-template.md) 骨架，**先過 template 內 lint checklist**，填入三輪實作 + Codex 完整 FAIL 理由原文 + executor 修改說明 + 主代理觀察的僵局類型）→ Run Log 追加 `[orchestrator] Termination triggered — impl 3-round FAIL` → **立刻停手**。INDEX.md 追加一行（結果欄寫 `⚠️ handoff`）。

### 步驟 9：經驗萃取（PASS 路徑專屬）

- **若 PASS**：`Agent(subagent_type="lesson-keeper", "本輪 PASS，請萃取經驗。計劃書：<路徑>，實作檔：<清單>，Codex 各輪理由：<原文>，Run Log：<wiki/runs/檔名>")`
- 從回傳擷取 `agentId` 記為 `<lesson_keeper_id>`，更新「Agent ID 索引」表
- 收「新增 N 條經驗」回報
- Run Log 追加 `[lesson-keeper] Spawn — id=<lesson_keeper_id>` / `[lesson-keeper] Return — N lessons`

### 步驟 10：收尾

- Run Log 追加 `[orchestrator] Workflow complete — total=Xh, plan-rounds=N, impl-rounds=M, lessons=K`
- 在 `wiki/runs/INDEX.md`（**cwd 相對**）追加一行（依 §3.5.5 表格格式）
- 回報使用者：「實作完成 + 本輪萃取 N 條經驗到 wiki/lessons.md」（N=0 時明說「本輪 0 條可萃取，符合預期」）

---

## Codex session 串接硬規則（1.0.4 介面適配）

- **沒有 `--resume <id>`，只有 `--resume-last`**：靠**序列保證**（advisor 階段全跑完才開 impl-validator）讓 `--resume-last` 在各 phase 內指對 session
- **絕不交錯**：advisor 4 輪中途絕不插任何 impl review；impl 8 輪中途絕不回頭問 advisor。交錯會讓下一次 `--resume-last` 接到錯誤 session 直接污染兩條歷史
- **過期處理**：`--resume-last` 失敗時重新 `--background`，把上輪 prompt 內容摘要塞回 focus text（算 1 輪）
- session-id 從 `--background` 回傳擷取記入 Run Log（**僅供 audit 與事後追溯**，不用於 `--resume`；檔案不 commit，無外洩風險）
- 觸發終局條件（advisor-stuck / handoff）→ 立刻停手，禁止為了「補一個 review」破壞序列

---

## 兩條交棒路徑都不自動觸發 lesson-keeper

handoff（impl 3 輪 FAIL）與 advisor-stuck（plan 3 輪無共識）都**不**呼叫 lesson-keeper。理由見規範 §3.3（路徑：`C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md`）：分歧本身可能正是 Codex 標準漂移或計劃規格不清，自動萃取容易把分歧當成原則。使用者裁決後若想萃取，請手動呼叫並指明萃取視角。

---

## Run Log 寫入禁忌

- **絕不寫敏感資料**：env vars、API key、使用者 PII 一律 `<redacted>` 取代；prompt 內容若含上述資料也要先擦掉
- **append-only**：絕不修改既有條目，只往下加
- **同步寫**：每次子代理回應 / Codex 回傳時立刻 append——不要批次留到最後（中途崩潰會失資料）

---

## 違規檢查清單（給未來主代理自我審查）

- [ ] 工作流啟動前有沒有先 `ToolSearch select:SendMessage` 確認工具可用？
- [ ] 步驟 1 啟動前有沒有確認 `plans/<功能>計劃.md` 已存在（**使用者寫好的**）？
- [ ] 有沒有不小心 Read 了實作檔程式碼？（除 handoff / advisor-stuck 整理對答歷史外，禁止）
- [ ] 有沒有不小心跑 `npm` / 測試指令？（一律由 executor 跑；`git add <plans/>` 屬於 Codex 餵檔機制例外）
- [ ] 有沒有不小心做**內容性判斷**？（架構好不好 / 程式碼對不對 / 建議有沒道理 / 誰立場對——一律 Codex / 子代理產出；參見「主代理角色」§可做 vs 不准做的清單）
- [ ] Codex prompt 注入有沒有用 v0.3 精簡版（只塞核心 + 引用檔路徑），還是退回 v0.2 貼全文？
- [ ] advisor 建議有沒有每條都帶 `[structural]` / `[non-structural]` 標籤？沒帶就請 Codex 補標
- [ ] 協商輪數有沒有按建議分級計（結構性 3 輪 / 非結構性 2 輪），還是統一一個上限？
- [ ] Codex 階段是否嚴格序列（advisor 全跑完才開 impl-validator），中途沒插入跨 phase 的 review？（1.0.4 `--resume-last` 介面要求）
- [ ] 每次 Agent spawn 後有沒有把 `agent_id` 立即填到 Run Log 上方索引表？
- [ ] 第二輪以後是用 `SendMessage(to=<id>, ...)` 還是又開了新 `Agent(...)`？（後者會失憶，禁止）
- [ ] Run Log 有沒有同步寫？（每個事件結束時就 append 一行，不要拖）
- [ ] 終局條件觸發後，handoff / advisor-stuck MD 寫完有沒有過 template 內的 lint checklist？
- [ ] 終局條件觸發後，有沒有立刻停手而非繼續修正？

任一條打不出勾 → 回頭修正，不要將錯就錯。
