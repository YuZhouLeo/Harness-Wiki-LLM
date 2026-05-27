# 多代理工作流 Skill 安裝包（給新機器 / 後來人重新部署用）

> **如果你（AI 或人類）正在另一台機器、想把這套工作流從零裝起來**：跟著本資料夾的 [安裝指南.md](安裝指南.md) 跑 3 步驟。
>
> **跟舊版安裝模式（複製 `.claude/.codex/plans/` 到專案根）的差異**：本版改為**全域 Skill** 模式——工作流檔案放在 `~/.claude/skills/multi-agent-dev/` 與 `~/.claude/agents/` 一份，所有專案共用，**不再複製進每個專案**。

---

## 30 秒看懂這套工作流在做什麼

把開發流程拆成六個明確角色，由主對話（orchestrator）純調度、子代理 / Codex 各自專責，每輪 PASS 終局自動沉澱經驗：

| 角色 | 由誰扮演 | 職責 |
|---|---|---|
| 主代理（orchestrator） | 你（主對話） | 純調度，不寫 code、不跑命令、不做技術判斷 |
| 規劃代理（planner） | Claude subagent | 把需求轉成計劃書 |
| 計劃顧問（plan-advisor） | Codex `--background` 諮詢模式 | 對計劃書提建議（諮詢，不是 PASS/FAIL） |
| 執行代理（executor） | Claude subagent | 依計劃書實作、跑 lint/test |
| 實作驗證者（impl-validator） | Codex `--background` 對抗模式 | 對實作 PASS / FAIL 守門 |
| 經驗代理（lesson-keeper） | Claude subagent | PASS 後把 Codex 駁回理由蒸餾為可遷移經驗 |

關鍵機制：
- **同身份重入修錯**：subagent 用 `Agent(...)` 第一次啟動後，後續輪次一律用 `SendMessage(to=<agentId>, ...)` 帶歷史續跑，避免重新讀檔失憶
- **跨模型驗證**：Codex（不同模型家族）審 Claude 的計劃與實作，消除同源誤差
- **計劃 / 實作雙 session 時序分離**：codex-plugin-cc 1.0.4 只有 `--resume-last`，工作流靠**序列保證**（advisor 階段全跑完才開 impl-validator）讓 `--resume-last` 各 phase 內指對 session；中途絕不交錯（介面適配詳見 [SKILL.md](../SKILL.md) §「Codex plugin 介面適配」）
- **3 輪上限自動交棒**：協商 3 輪無共識 → advisor-stuck MD；驗證 3 輪 FAIL → handoff MD；交棒後立刻停手等使用者裁決
- **經驗回流**：每輪 PASS 由 lesson-keeper 寫進**各專案的** `wiki/lessons.md`，下一輪 planner / executor 開工前先 grep

完整設計理由 → [多代理協作工作流計劃.md](../spec/多代理協作工作流計劃.md)（~990 行）+ [adr/0002](../adr/0002-codex-as-validator-agent.md) ~ [0006](../adr/0006-max-retry-three-rounds.md)。

---

## 全域 vs 各專案資產分布

| 類別 | 位置 | 說明 |
|---|---|---|
| 工作流自身規範 / prompts / 模板 / ADR 0001-0006 | `~/.claude/skills/multi-agent-dev/`（**全域共享**） | 跨專案一份，由本 skill 提供 |
| 3 個 user-level subagent | `~/.claude/agents/{planner,executor,lesson-keeper}.md`（**全域共享**） | `Agent(subagent_type=...)` 直接可用 |
| SendMessage 實驗 flag | `~/.claude/settings.json` 頂層 env（**全域**） | 全機共享 |
| 各專案累積的經驗 | `<project>/wiki/lessons.md`（**各專案**） | 不放全域；首次寫入時 lesson-keeper 自動建骨架 |
| 各專案 Run Log | `<project>/wiki/runs/<時間戳>-*.md` + `INDEX.md`（**各專案**） | 同上；單檔不 commit（`.gitignore` 規則） |
| 各專案計劃書 | `<project>/plans/<功能>計劃.md`（**各專案**） | planner 寫入位置 |
| 各專案 invariants | `<project>/docs/invariants.md`（**各專案**） | 可選；無則 fallback 為 AGENTS.md |
| 各專案業務 ADR (0007+) | `<project>/docs/ADR/0007+`（**各專案**） | 工作流 ADR 0001-0006 已在全域，**不要複製進專案** |

---

## 安裝後的觸發方式

在任何專案的 Claude Code 主對話：

```
啟動多代理開發：<你的需求>
```

或顯式：

```
/multi-agent-dev <你的需求>
```

主代理會自動載入本 skill 的 [SKILL.md](../SKILL.md)（11 步調度流程），不需要專案內有任何工作流檔案。

---

## 資料夾結構（本安裝包內含）

```
~/.claude/skills/multi-agent-dev/
├── SKILL.md                                # 主入口 + frontmatter 觸發描述 + 11 步調度
├── codex-prompts/                          # Codex 兩種模式 prompt
│   ├── advisor-prompt.md
│   └── impl-validator-prompt.md
├── spec/                                   # 設計規範
│   ├── 多代理協作工作流計劃.md             #   主規範文件（~990 行）
│   └── SendMessage子代理續用使用指南.md    #   SendMessage 啟用與排錯
├── templates/                              # 交棒模板
│   ├── advisor-stuck-template.md
│   └── handoff-template.md
├── adr/                                    # 工作流自身 ADR（0001-0006）
│   ├── README.md
│   ├── 0001-record-architecture-decisions.md
│   ├── 0002-codex-as-validator-agent.md
│   ├── 0003-split-plan-advisor-and-impl-validator.md
│   ├── 0004-lessons-three-writing-principles.md
│   ├── 0005-run-log-md-timeline-format.md
│   └── 0006-max-retry-three-rounds.md
└── installation/                           # ★ 你正在讀這裡
    ├── README.md                           #   本檔
    ├── 安裝指南.md                         #   3 步驟安裝
    ├── invariants.md.範本                  #   給各專案複製到 docs/invariants.md 的範本
    └── 整合片段/                           #   給各專案合進既有檔案的片段
        ├── AGENTS.md.片段                  #     合進 <project>/AGENTS.md
        ├── settings.json.片段              #     merge 到 ~/.claude/settings.json 頂層
        └── .gitignore.片段                 #     合進 <project>/.gitignore

~/.claude/agents/                           # user-level subagent（與 skill 同層平等）
├── planner.md
├── executor.md
└── lesson-keeper.md
```
