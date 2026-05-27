# ADR-0005: Run Log 採 MD timeline 格式

- 狀態：Accepted
- 日期：2026-05-06

## 脈絡

[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §3.5 引入 Run Log 系統，補上工作流缺乏「跨代理活動日誌」的缺口。沒有 Run Log 的話：

- 工作流出問題時，使用者只能從 handoff / advisor-stuck MD 反推，缺前期過程
- lesson-keeper 必須完整翻過所有 SendMessage 摘要才能萃取經驗，沒有彙整版可參考
- 跨任務查找「上次類似功能怎麼跑的」沒有索引可循
- 主代理對自己「這輪做過什麼」沒有可回顧的事實基準

需要決定 Run Log 的**檔案格式**。候選方案各有取捨：

- **ADR-style（多 section、重格式）**：適合靜態決策，不適合「事件流」。每筆事件要分段、分標題，寫起來很重，讀起來反而難看出時間線
- **JSON Lines**：機器可讀、結構嚴謹，但人類眼睛掃不到「14:32 → 14:35」的時間關係；要看必須跑 jq / 寫腳本，違背「使用者隨時可讀」的目標
- **單一 rolling log（所有 run 共用一檔）**：易混雜、難 grep 特定 run、檔案無限長
- **MD timeline（一行一事件、append-only、per-run 一檔）**：人類可讀、grep 友善、每 run 獨立、append-only 適合事件流本質

事件流的本質是「一行一事件、按時間排序、不修改既有條目」。這個本質正好對應 MD bullet list 或固定欄位的 plain text，**不需要 ADR 那種多 section 結構**。

## 決策

採用 **MD timeline 格式**：一行一事件、append-only、per-run 一檔。

### 檔案結構

```
wiki/runs/
├── INDEX.md                                # 所有 run 的索引（commit）
├── 20260506-1432-台股-daily-summary.md     # 單一 run 完整時間線（不 commit）
├── 20260506-1610-watchlist-export.md
└── ...
```

- 命名：`<YYYYMMDD-HHmm>-<功能 slug>.md`，時間以工作流啟動為準
- INDEX.md 每次 run 結束時由主代理追加一行

### 條目格式

每條一行，固定 4 欄：

```
[HH:mm:ss] [actor] action — result/payload
```

`actor` 為下列之一（與 [plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §2 角色一致）：

- `orchestrator` — 主代理
- `planner` / `executor` / `lesson-keeper` — Claude subagent
- `plan-advisor` / `impl-validator` — Codex 兩種模式

範例：

```text
[14:32:01] [orchestrator] Workflow start — feature="台股 daily summary"
[14:32:05] [planner] Spawn — id=plan-001
[14:35:42] [planner] Return — plans/台股-daily-summary計劃.md (87 lines)
[14:35:50] [plan-advisor] Codex review --background — session=advisor-001
[14:38:12] [plan-advisor] Return — 4 suggestions (architecture x3, tooling x1)
...
```

### 寫入規則

1. **唯一寫入者：主代理**。子代理不直接寫日誌；它們回傳的摘要由主代理轉成日誌條目
2. **append-only**：絕不修改既有條目，只往下加；避免事後改寫導致審計失真
3. **同步寫**：每次主代理收到子代理回應 / 呼叫 Codex 完成，**立刻**追加一行（不要批次留到最後，否則中途崩潰會失資料）
4. **不准寫敏感資料**：env vars、API key、使用者 PII 一律以 `<redacted>` 取代；prompt 內容若含上述資料也要先擦掉

### Git 政策

- **commit**：`wiki/runs/INDEX.md`（只記時間 / 任務 / 結果統計，無敏感內容）
- **不 commit**（加進 `.gitignore`）：`wiki/runs/[0-9]*.md`（單一 run 完整日誌，可能含 prompt 細節）

詳細規範見 [plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §3.5。

評估過的替代：

- **ADR-style（多 section、重格式）**：每筆事件分段分標題，寫起來重、讀起來反而看不出時間線。**否決**——格式與內容性質不匹配
- **JSON Lines**：機器可讀，但人類掃時間關係困難；不符合「使用者隨時可開來看」的設計。**否決**
- **單一 rolling log（所有 run 一檔）**：grep 不到特定 run、檔案無限長、難以歸檔。**否決**
- **記憶體只記、不落地**：工作流崩潰即失資料。**否決**——無法滿足 [plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §3.5.1 的核心動機
- **事件 → 寫資料庫（SQLite / IndexedDB）**：查詢能力強，但 OpenStock 目前沒有 local DB；新增依賴與後續維護成本。**否決**——殺雞用牛刀

## 後果

**好處**

- 人類可讀：使用者隨時可 `cat wiki/runs/<檔名>` 看時間線，不需要工具
- grep 友善：固定 4 欄格式，`grep "[planner]"`、`grep "FAIL"` 命中精準
- append-only 適合事件本質：審計軌跡不會被事後改寫
- per-run 一檔：歸檔、清理、`.gitignore` 規則簡單
- INDEX.md 提供跨 run 的快速索引：使用者想找「上次類似功能怎麼跑的」一目了然

**壞處**

- **解析需簡單腳本**：未來若要對 run log 做統計分析（例如「平均 plan 輪數」），需自行寫 grep / awk（可接受——目前需求只是「人類可讀 + lesson-keeper 全文讀」，不需結構化查詢）
- **版本演進時舊 log 不相容**：若未來改格式（增 / 減欄位），舊 log 不再符合新解析腳本（緩解：欄位數固定 4，演進時優先選擇 append-compatible 變更；真要破格時保留舊版解析邏輯）
- **`wiki/runs/` 檔案累積過多**：跑多了會變幾百個檔的墳場（緩解：[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §7 風險表第 17 條，命名含日期方便 glob 清理；Phase 2 觀察平均大小，必要時改月度封存到 `.archive/`）
- **同步寫的 I/O 成本**：每個事件都要 Edit 一次（可接受——append 操作很輕，且可靠性勝過效能）

**需要持續觀察**

- 平均一個 run log 大小：Phase 2 結束時統計，決定是否需要分檔或壓縮（[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §3.5.7）
- 多少 % 的 run 真的被人類事後讀取：若 < 10%，考慮停止寫入或改為精簡版
- lesson-keeper 用了 run log 之後，萃取品質有沒有實質提升：vs 只看 SendMessage 摘要的對照組
- 敏感資料外漏事故率：若有任何一次 PII / API key 沒被 redact 出現在 commit 的 INDEX.md，本 ADR 必須立刻補充更嚴格的 redaction 規範
