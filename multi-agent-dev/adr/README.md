# Architecture Decision Records

> **本目錄（`~/.claude/skills/multi-agent-dev/adr/`）= 全域工作流 ADR**（編號從 0001 起，可成長）。
> 各專案業務 ADR 放在各專案的 `docs/ADR/`，編號獨立計（**建議從 0007 起算**避免目視混淆，但物理上兩目錄不會碰撞）。
> 規則整理見本檔末尾「§ 雙層 ADR 配置」。

本目錄存放本專案的**架構決策紀錄（Architecture Decision Records, ADR）**，用來保留「**為什麼**這樣做」的歷史脈絡。

## 什麼時候要寫 ADR

當決策符合下列任一條件，就該開一份 ADR：

- 影響跨多個模組或目錄（例如：選擇 ORM、選擇背景任務系統、選擇 auth 方案）
- 一旦上線後翻盤成本高（資料庫格式、API 介面契約、使用者資料模型）
- 評估過多個替代方案，且選擇的理由非顯而易見
- 引入新的外部服務或執行階段依賴

**不需要寫 ADR 的情境**：bug fix、單一檔案重構、UI 樣式調整、依賴版本升級、純文件更新。

## 格式

採用 [Michael Nygard ADR](https://github.com/joelparkerhenderson/architecture-decision-record) 簡化版：

```markdown
# ADR-NNNN: 標題

- 狀態：Accepted | Superseded by ADR-XXXX | Deprecated
- 日期：YYYY-MM-DD

## 脈絡
（當時面對的問題與限制）

## 決策
（選了什麼、評估過哪些替代、為何這樣選）

## 後果
（好的、壞的、需要持續觀察的）
```

## 編號規則

- 從 `0001` 開始，**不准跳號**
- 檔名格式：`NNNN-kebab-case-title.md`，例如 `0002-choose-better-auth.md`
- 一旦合併進 main 就**視為不可變**——要推翻就寫一份新 ADR，並把舊 ADR 狀態改為 `Superseded by ADR-XXXX`，**不要直接改舊 ADR 的內容**

## 既有 ADR（全域工作流 ADR）

> 下列 ADR 是**多代理工作流自身的設計理由**，由 `multi-agent-dev` skill 全域共享，所有專案的 planner / impl-validator 引用時走絕對路徑 `C:/Users/yuzho/.claude/skills/multi-agent-dev/adr/`。
> 它們記錄的是工作流本身的設計決策，不是任一專案的業務決策。

| 編號 | 標題 | 狀態 |
|---|---|---|
| [0001](0001-record-architecture-decisions.md) | 採用 ADR 機制本身 | Accepted |
| [0002](0002-codex-as-validator-agent.md) | 採用 Codex 作為驗證代理 | Accepted |
| [0003](0003-split-plan-advisor-and-impl-validator.md) | 驗證代理拆為 Plan-Advisor 與 Impl-Validator | Accepted（部分被 ADR-0007 修訂） |
| [0004](0004-lessons-three-writing-principles.md) | 經驗資料庫採三大寫入原則 | Accepted |
| [0005](0005-run-log-md-timeline-format.md) | Run Log 採 MD timeline 格式 | Accepted |
| [0006](0006-max-retry-three-rounds.md) | max retry = 3（兩個驗證階段共用） | Accepted（部分被 ADR-0007 修訂） |
| [0007](0007-karpathy-baseline-tiered-rounds-slim-prompts.md) | Karpathy 基線 + 分級協商上限 + Prompt 注入精簡 | Accepted |

新專案要寫自己的業務決策 ADR 時，**建議從 0007 起算**（純為避免目視混淆；物理上兩目錄不會碰撞）。

## 雙層 ADR 配置

| 範疇 | 位置 | 編號 | 引用方式 |
|---|---|---|---|
| **全域工作流自身** | `C:/Users/yuzho/.claude/skills/multi-agent-dev/adr/` | 0001 起，**可成長** | planner / impl-validator 走絕對路徑 |
| **各專案業務決策** | `<project>/docs/ADR/` | 建議 0007 起 | planner / impl-validator 走 cwd 相對路徑 |

**絕不混用**：
- 全域層只放工作流自身設計決策（advisor / executor / Codex session 邏輯等）；不放任一專案的業務 ADR
- 各專案層只放該專案的業務決策（DB schema、API 契約等）；不複製全域工作流 ADR

物理上兩目錄不會碰撞——同樣編號（例如兩邊都有 0007）的兩份 ADR 可共存。「建議 0007 起」純為避免目視混淆，不是硬性鎖死。

新專案首次安裝多代理工作流時，**不需要**把全域 ADR 複製進 `<project>/docs/ADR/`——planner / impl-validator 已被改寫為走全域絕對路徑。各專案的 `docs/ADR/` 只放自家業務 ADR 與 `README.md` meta。
