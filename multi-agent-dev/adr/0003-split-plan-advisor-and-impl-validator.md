# ADR-0003: 驗證代理拆為 Plan-Advisor 與 Impl-Validator

- 狀態：Accepted
- 日期：2026-05-06

## 脈絡

[ADR-0002](0002-codex-as-validator-agent.md) 確認用 Codex 擔任驗證代理。但驗證在工作流的兩個階段扮演的角色其實**不一樣**：

| 階段 | 互動模式 | 終局形式 | 對話節奏 |
|---|---|---|---|
| 計劃階段 | **諮詢協商**：顧問提建議，規劃代理可反駁，多輪共識 | 共識達成（採納或顧問接受不採納） | 來回討論、可堅持、可退讓 |
| 實作階段 | **PASS / FAIL 守門**：對抗式 review、找碴心態 | 二擇一裁決 | 嚴格、不放水、不妥協 |

如果**共用一個 logical role**（同 prompt template、同 session），會出現以下問題：

- **角色錯亂**：諮詢時 Codex 進入「找碴守門」心態 → 規劃代理收到一堆 PASS/FAIL，協商根本不存在
- **session 污染**：實作驗證 resume 到 advisor session → Codex 會看到「請提建議」歷史，對實作放水或自我矛盾
- **prompt template 混亂**：要在同一份 prompt 同時容納「諮詢者請保持開放」和「守門員請嚴格」，邏輯打架
- **終局規則無法統一**：協商的「3 輪無共識」是寫 advisor-stuck MD；驗證的「3 輪 FAIL」是寫 handoff MD——兩者形式不同、處理路徑不同，不能共用單一終局函式

實際運行中，「共用」會以最壞的方式表現：advisor session 累積對抗對話 → 第二次規劃時 Codex 帶著上次的對抗痕跡批計劃；或者 impl session 帶著諮詢心態 → review 變成「可以更好」的建議清單而非守門。

## 決策

驗證代理拆為 **兩個獨立的 logical role**，共用 Codex 後端但 prompt template / session 完全分離：

### 邊界規範

| 項目 | Plan-Advisor（諮詢） | Impl-Validator（守門） |
|---|---|---|
| 角色定位 | 諮詢者，提建議 | 守門員，PASS / FAIL 裁決 |
| Slash command | `/codex:review` | `/codex:adversarial-review` |
| Prompt template | [.codex/advisor-prompt.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/codex-prompts/advisor-prompt.md) | [.codex/impl-validator-prompt.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/codex-prompts/impl-validator-prompt.md) |
| Session 啟動 | `--background` 取得 advisor-session-id | `--background` 取得 impl-session-id |
| Session 接續 | `--resume <advisor-session-id>` | `--resume <impl-session-id>` |
| 終局規則 | 3 輪無共識 → advisor-stuck MD | 3 輪 FAIL → handoff MD |
| 觸發 lesson-keeper | 否（[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §3.3） | PASS 時觸發 |

### 硬規則

1. **session 絕對不可互 resume**：advisor session 不應收到 lint/test 輸出；impl session 不應收到「請提建議」prompt
2. **prompt template 物理分離**：兩份檔在 `.codex/` 各佔一檔，主代理調度時複製對應內容
3. **無 Claude subagent 檔**：兩個角色都不在 `.claude/agents/` 建立檔案——驗證者不是 Claude subagent
4. **主代理職責**：分別記錄並維護 advisor-session-id 與 impl-session-id 兩個變數，每輪選對 `--resume` 目標

評估過的替代：

- **單一 validator 角色（諮詢與守門合一）**：上述「角色錯亂、session 污染、prompt 混亂、終局無法統一」全部命中。**否決**——這是要避免的反模式
- **兩個獨立 Claude subagent 各自包一個 Codex 後端**：增加 Claude 層次的調度複雜度，且 Claude 主代理需要透過 SendMessage 跟 Claude subagent 交談，而 Claude subagent 再呼叫 Codex——多一層中介、多一層 token、多一層失效點。**否決**——直接讓主代理同時管兩個 Codex session 更簡單
- **Plan-Advisor 改用 Claude（非 Codex）**：失去「跨模型驗證消除同源誤差」的核心價值（[ADR-0002](0002-codex-as-validator-agent.md)）。**否決**——規劃階段也要跨模型視角
- **共用 session 但每階段切換 prompt**：實作上需要中間插入「請忘掉上一階段的 prompt」指令；模型實際遵守程度未知。**否決**——session 的「記憶」就是要被利用的，硬切換違反設計

## 後果

**好處**

- 兩個角色各自有清楚的互動模式，prompt template 簡潔不需要兼容
- 終局規則對應各自階段（協商 vs 對抗），主代理判斷邏輯清晰
- 一個 session 出問題（失效、污染）不會拖累另一個

**壞處**

- **要維護兩份 prompt template**：[advisor-prompt.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/codex-prompts/advisor-prompt.md) 與 [impl-validator-prompt.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/codex-prompts/impl-validator-prompt.md)；模板演進時要同時更新（緩解：兩份都引用各專案的 `AGENTS.md` §Invariants 與 `docs/conventions/`（cwd 相對），共通規範集中管理）
- **session-id 弄混的風險**：主代理拿著 advisor-session-id 跑去 resume impl session（緩解：[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §4.2.4 共通禁止項；§7 風險表第 4 條；主代理調度提示明寫「impl 階段一律 `--background` 新 session，不准 resume advisor」）
- **token 成本提升**：一次工作流 Codex 兩個 session 各自累積上下文（緩解：advisor 通常 1–2 輪共識即達成，session 不長）

**需要持續觀察**

- 兩個 session 的「實際分離度」：Phase 2 觀察是否真的有「advisor session 滲入對抗心態」或「impl session 滲入諮詢心態」的現象
- 兩份 prompt template 的維護成本：若演進時頻繁要同步更新，考慮抽出共通段落為 `.codex/_shared.md` 由兩份 import
- session-id 弄混的實際發生率：若超過 5%（每 20 次工作流出一次），需在主代理調度提示加更強的 sanity check
