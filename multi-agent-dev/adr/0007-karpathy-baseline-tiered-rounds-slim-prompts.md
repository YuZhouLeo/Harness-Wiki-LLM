# ADR-0007: Karpathy 基線 + 分級協商上限 + Prompt 注入精簡

- 狀態：Accepted
- 日期：2026-05-25
- 取代/被取代：部分修訂 [ADR-0003](0003-split-plan-advisor-and-impl-validator.md)（advisor 審查面向新增工作拆分維度）、部分修訂 [ADR-0006](0006-max-retry-three-rounds.md)（協商輪數從統一 3 輪 → 分級 2/3 輪）

## 脈絡

工作流 v0.2 跑了若干輪後，使用者觀察出三個結構性問題：

### 問題 1：審查標準不一致

advisor / impl-validator 都用「自由心證」審 + 各自 prompt 內的散裝原則。沒有第一性基線可錨定。同一份計劃書，advisor 心情好寫法不同；不同人讀 impl-validator-prompt.md 也會解讀分歧。

### 問題 2：協商輪數一刀切，浪費在風格爭議

[ADR-0006](0006-max-retry-three-rounds.md) 設兩個驗證階段共用 max retry = 3。實務上：

- **結構性建議**（架構分層、套件選擇、相依性、安全）3 輪剛好足夠
- **非結構性建議**（命名、文字精細度、註解品質）2 輪內如果不收斂多半是純風格分歧，第 3 輪是浪費

統一 3 輪上限導致：
- 非結構性建議經常拖到 3 輪才放棄 → token 浪費 + 拖節奏
- 觸發 advisor-stuck 後使用者一看：「這也叫 stuck？就命名嘛」→ template 信號被稀釋

### 問題 3：Codex prompt 全文每 phase 重塞

v0.2 步驟 2 / 步驟 7 把 advisor-prompt.md / impl-validator-prompt.md 全文塞進 `[focus] ...` focus text。兩份 prompt 合計約 150 行 / 約 2.5K tokens。雖然 `--resume-last` 後續輪次不重塞，但兩個 phase 各塞一次是固定成本。

### 問題 4：主代理「不准做技術判斷」邊界模糊

v0.2 寫「不准做技術判斷」但實務上主代理必須做：
- 「PASS 字串是否出現」← 是判斷
- 「輪數是否達上限」← 是判斷
- 「Codex 回應該轉給 planner 還是 executor」← 是判斷

模糊邊界導致兩種失誤：(a) 主代理過度自我約束，連結構性記帳都不敢做 → 流程停滯；(b) 主代理過度延伸，連「這個架構好不好」都自己評 → 違反零生產原則。

### 問題 5：handoff / advisor-stuck template 無填寫驗收

v0.2 的 template 是骨架字串。主代理寫完直接交棒，沒有 lint 步驟，常見漏填：
- `<原文>` 預留字串沒填就交棒
- 「下一步選項」被預先勾選一項
- Run Log 連結寫成絕對路徑

使用者收到不合格 MD 還要回頭補。

## 決策

### 決策 1：審查基線改用 `/karpathy-guidelines` skill

advisor 與 impl-validator 兩個 prompt 都新增「審查基線」段，明示以 [karpathy-guidelines/SKILL.md](C:/Users/yuzho/.claude/skills/karpathy-guidelines/SKILL.md) 的四條原則為第一性錨：

1. Think Before Coding（surface assumptions / tradeoffs）
2. Simplicity First（最小程式碼解問題、無投機式擴張）
3. Surgical Changes（只動該動的、不順手「改善」相鄰）
4. Goal-Driven Execution（可驗證成功標準）

Codex 每輪審查前先 read 該 skill 全文，把四條原則當作對齊基準。違反任一條 → advisor 提建議 / impl-validator FAIL。

### 決策 2：協商輪數分級

advisor 每條建議**必須**標 `[structural]` 或 `[non-structural]` 標籤：

| 標籤 | 上限 | 達上限後 |
|---|---|---|
| `[non-structural]` | 2 輪 | 軟性終局：主代理自動視為「顧問接受規劃方案」，繼續往下走，**不寫 advisor-stuck MD** |
| `[structural]` | 3 輪 | 硬性終局：寫 advisor-stuck MD 交棒使用者裁決 |

判斷準則（advisor 自我標）：
- `[structural]`：影響架構分層、套件選擇、可拆分性、安全性、資料完整性、相容性、ADR/invariants 違反
- `[non-structural]`：純命名、純風格、單一變數的小重構、註解品質、文字精細度

判斷不清時**標 `[structural]`** 從嚴（多給協商機會）。

### 決策 3：Prompt 注入精簡

步驟 2 / 步驟 7 的 focus text 改為：
- **塞**：角色定位、四大面向 / 5 條檢查項、嚴重度標籤、輸出格式、必讀清單路徑
- **不塞**：行為哲學、終局自律、禁止清單（這些不變的，Codex 自己 read 該 prompt 檔絕對路徑取得）

兩個 prompt 檔的開頭「用法」段示範新呼叫方式，並保留全文作為 Codex read 用的 reference。

每輪節省約 1.5K tokens（兩 phase 各約 750 tokens 浪費）。

### 決策 4：主代理判斷邊界明確化

把「不准做技術判斷」拆成兩類：

**可做：結構性判斷**（語法 / 字面層次，不涉及對錯）
- PASS / FAIL 字串、輪數計數、`[structural]` 標籤擷取、檔案路徑 / agent_id / session_id 字串擷取、終局條件命中

**不准做：內容性判斷**（對錯 / 好壞 / 該不該）
- 架構合理性、程式碼正確性、計劃書完整性、建議有沒道理、誰立場對

清單寫進 SKILL.md「主代理角色」章節。

### 決策 5：Template 填寫 lint checklist

[advisor-stuck-template.md](../templates/advisor-stuck-template.md) 與 [handoff-template.md](../templates/handoff-template.md) 開頭都加一段「填寫檢查清單」，主代理寫完後逐項自我審查：

- 預留字串（`<原文>` / `<Codex 回傳全文>`）是否還在 → 在就是沒填
- 「下一步選項」是否預先勾選 → 不准
- Run Log 連結是否相對路徑 → 不准絕對路徑
- 檔案命名是否符合 `plans/<功能名>-<類型>-<時間戳>.md` 規格

任一條打不出勾 → 回頭補，不交棒。

## 評估過的替代

### 決策 1（審查基線）替代

- **不引入外部 skill，把 Karpathy 原則直接抄進兩個 prompt** → 否決：複製貼上會漂移；Karpathy 原則若未來更新，兩個 prompt 不會同步
- **用 SOLID / Clean Code 等更傳統的基線** → 否決：太抽象，Codex 解讀分歧大；Karpathy 四條夠具體可審

### 決策 2（分級上限）替代

- **維持統一 3 輪** → 否決：問題 2 已說明
- **結構性 5 輪 / 非結構性 2 輪**（給結構性更多時間） → 否決：3 輪已是 ADR-0006 守緊起步的決定；Phase 2 觀察後若需要再放寬
- **每條建議獨立計輪、最後合併**（複雜計輪邏輯） → 否決：複雜化主代理；採用「同一輪內標籤分開計、達上限的標籤對應條目套對應終局」即可

### 決策 3（prompt 精簡）替代

- **完全不塞 prompt，Codex 全靠 read** → 否決：Codex 不見得會主動 read；focus text 至少要點明「請 read X」+ 提供輸出格式錨點
- **把 prompt 檔搬到專案 working tree** → 否決：兩 prompt 是工作流自身資產，不該汙染各專案 repo
- **保留全文塞入** → 否決：問題 3 已說明的 token 浪費

### 決策 4（判斷邊界）替代

- **完全禁止主代理任何判斷，連 PASS 字串都不能讀** → 否決：流程無法推進，PASS / FAIL 字串擷取是 orchestration 必要動作
- **完全允許主代理做技術判斷** → 否決：違反零生產原則，主代理失去客觀仲裁者角色

### 決策 5（template lint）替代

- **不加 lint，靠主代理小心** → 否決：實證已知會漏填
- **寫一個 markdown linter 腳本檢查 template 填寫** → 否決：過度工程；checklist + 主代理自我審查夠用
- **每寫一段就停手等使用者確認** → 否決：違反「立刻停手交棒」原則，且使用者要的是完整 MD 一次看完

## 後果

**好處**

- 審查口徑可複現：所有 advisor / impl-validator 對齊 Karpathy 四原則，跨輪次 / 跨任務一致
- 非結構性爭議不再吃 advisor-stuck 配額：軟性終局自動收，advisor-stuck MD 信號純度提升
- 每 run 節省約 1.5K tokens
- 主代理失誤類型可定義：違反邊界清單 = 違規；不必再爭辯「這算不算技術判斷」
- handoff / advisor-stuck MD 品質提升：使用者一打開就是完整可裁決的版本，不必回頭補

**壞處**

- advisor 多了一個責任：每條建議標標籤。標錯（把結構性建議標成非結構性）會讓該建議第 2 輪就被軟性接受，掩蓋真問題。緩解：prompt 寫「判斷不清時標 `[structural]`」從嚴
- prompt 引用路徑機制依賴 Codex 真的會 read：若 Codex 因為 working tree 內無此檔而拒讀，需要降級回 v0.2 全文塞入。Phase 2 需觀察
- template lint checklist 增加主代理工作量：但這本來就是該做的，只是過去沒明示

**需要持續觀察**

- 結構性 / 非結構性的標籤分布：若 ≥ 80% 都被標 `[structural]`，表示「判斷不清從嚴」規則被過度應用，標籤失去區分價值——需在 advisor-prompt 加更具體的歸類例子
- 非結構性軟性終局後，使用者實際是否同意「自動接受」：若使用者經常事後翻案說「那條我其實要採納」→ 軟性終局太激進，需改成「達上限後問使用者一次」
- prompt 引用路徑後 Codex 的審查深度：與 v0.2 全文塞入相比，若 finding 質量明顯下降（例如 INV-XX 引用率掉超過 30%），降級回全文塞入
- ADR-0006 的 Phase 2 觀察條件（前 5 任務 ≥ 30% 跑滿 3 輪）：本 ADR 引入分級後重新計算——應該分開觀察 `[structural]` 與 `[non-structural]` 的收斂分布

## 相關變更

本 ADR 對應的具體檔案變更：

- [SKILL.md](../SKILL.md) — 步驟 1 重定義（planner 改為讀使用者計劃書 → 輸出工作拆分）、主代理角色重寫、步驟 5 分級輪數、步驟 2/7 prompt 注入簡化、違規檢查清單擴充
- [codex-prompts/advisor-prompt.md](../codex-prompts/advisor-prompt.md) — 新增 §審查基線、新增工作拆分審查面向、新增 `[structural]` 標籤要求
- [codex-prompts/impl-validator-prompt.md](../codex-prompts/impl-validator-prompt.md) — 新增 §審查基線、新增檢查項 5（從簡 + 低負載）、明示穩定性由 Codex 內建涵蓋
- [templates/advisor-stuck-template.md](../templates/advisor-stuck-template.md) — 新增填寫 lint checklist、條目格式加 `[structural|non-structural]` 標籤欄
- [templates/handoff-template.md](../templates/handoff-template.md) — 新增填寫 lint checklist、§2 同時引用計劃書 + 工作拆分檔
- [templates/user-plan-template.md](../templates/user-plan-template.md) — **新建**，定義使用者計劃書範本
