# SendMessage 子代理續用使用指南

> 解決問題：主代理無法在第二輪呼叫**同一個** subagent 進行修改 —— 用 `Agent(...)` 重新呼叫會開新實例、失去歷史。
> 解法：啟用 `agent teams` 實驗功能後，改用 `SendMessage` 對先前 subagent 的 `agentId` 投遞訊息，即可帶完整歷史續跑。

來源：[plans/custom subagents.md:742-760](custom%20subagents.md#L742-L760)（Resume subagents 段）。

---

## 1. 啟用條件

### 1.1 必須設置環境變數

```
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

**重點**：此變數必須在 Claude Code 啟動**之前**設好。session 開始後再設無效，會看到：
- `ToolSearch select:SendMessage` 回 `No matching deferred tools found`
- `Agent(...)` 雖然能正常用，但你拿到 agentId 也呼叫不了 SendMessage

### 1.2 兩種設置方法

#### 方法 A：寫進 `.claude/settings.json`（建議，跟著專案走）

在專案根目錄的 [.claude/settings.json](../.claude/settings.json) **頂層**新增 `env` 區塊（⚠️ 不要放進 `permissions` 內）：

```json
{
  "permissions": {
    "allow": [ ... 既有內容 ... ]
  },
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

**好消息**：存檔當下 Claude Code 會熱載入新 env，**不需要重啟**。可立即用 `ToolSearch select:SendMessage` 驗證工具是否上線。

#### 方法 B：shell 啟動時 export（適合臨時測試）

```bash
# Git Bash / WSL
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
claude

# PowerShell
$env:CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS = "1"
claude
```

### 1.3 驗證已啟用

啟用後，在新 session 內執行：

```
ToolSearch query="select:SendMessage" max_results=1
```

若看到 SendMessage 的完整 schema 回傳即成功；仍回 `No matching deferred tools found` 表示變數未生效。

---

## 2. 使用流程

### 2.1 三步驟基本範式

```
[1] 主代理 → Agent(...)        ⇒ 拿到 result + agentId
[2] 主代理 → 把 agentId 記下來  ⇒ 後續同一輪流程都引用它
[3] 主代理 → SendMessage(...)   ⇒ 同一個 subagent 帶歷史續跑
```

### 2.2 第一步：呼叫 Agent 並擷取 agentId

```
Agent(
  description="實作 X 模組",
  subagent_type="executor",
  prompt="依 plans/xxx.md 實作 ..."
)
```

回傳格式（**已實測**）：

```
<agent 的 final result 內容>
agentId: ae4af9c7ebbbc3bba (use SendMessage with to: 'ae4af9c7ebbbc3bba' to continue this agent)
<usage>total_tokens: 16971 ...</usage>
```

主代理必須：
- 把 `agentId` 字串保存下來（寫進 plan 文件、todo list、或 conversation context 都行）
- 該 ID 在當前 session 永遠有效，subagent 已停止也能透過 SendMessage 喚醒

### 2.3 第二步：用 SendMessage 續用

實測可用的呼叫格式（**已驗證**）：

```
SendMessage(
  to:      "ae4af9c7ebbbc3bba",           # 上一步保存的 agentId
  summary: "5–10 word preview",            # UI 預覽用，必填
  message: "<新指示，例如 codex 駁回理由 + 修正方向>"
)
```

回傳格式：
```json
{
  "success": true,
  "message": "Agent \"<id>\" had no active task; resumed from transcript in the background with your message. You'll be notified when it finishes."
}
```

行為（已實測）：
- subagent 帶**完整歷史**（先前的 tool 呼叫、reasoning、檔案讀取結果）續跑
- 若 subagent 已 stopped，自動在**背景** resume，不需要新的 `Agent(...)` 呼叫
- 主代理不被阻塞，繼續做其他事
- subagent 完成時透過 system task-notification 推送 `<status>completed</status>` 與 `<result>...</result>`
- 結束後可再次 SendMessage 投遞下一輪指示給同一 agentId

### 2.4 取得歷史 transcript（除錯用）

每個 subagent 的完整 transcript 存於：

```
~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl
```

可手動 Read 這個檔案查看 subagent 內部的所有對話、tool call、compaction 紀錄。

---

## 3. 整合到本專案多代理流程

本專案 [.claude/orchestrator.md](../.claude/orchestrator.md) 11 步流程中，**executor 被 codex 駁回後重新進場修改**正是 SendMessage 的最佳使用場景。建議改寫如下：

### 改寫前（會失憶）
```
Round N:    Agent(executor, 完整 prompt)        → result_N, agentId_N (棄用)
Round N+1:  Agent(executor, 完整 prompt + 駁回理由)  ← 開新實例，重新讀檔、重新理解
```

### 改寫後（保留上下文）
```
Round N:    Agent(executor, 完整 prompt)        → result_N, agentId_N (保存)
Round N+1:  SendMessage(to=agentId_N,
                       message="codex 駁回理由：...，請修正：...")
            ↑ 同一 executor 帶歷史續跑，不重讀檔、token 省、語境連貫
```

### orchestrator.md 修改建議

在第 N 輪 executor 完成後加一行：「**保存 agentId 到 wiki/runs/<run-id>.md 的 executor_agent_id 欄位**」，第 N+1 輪改用 SendMessage 而非 Agent 呼叫。

---

## 4. 限制與注意事項

| 限制 | 說明 |
|------|------|
| 跨 session 失效 | agentId 只在當前 Claude Code session 內有效；重啟後即使 transcript 還在也無法 SendMessage 喚醒 |
| 仍無法巢狀 | subagent 內部依然不能 spawn 子-subagent；SendMessage 只是「續用」，不是「新生」 |
| 背景化行為 | stopped subagent 收到 SendMessage 會在背景 resume，不阻塞主對話 |
| 權限繼承 | resume 時權限 / 工具 / model 與第一次 spawn 時相同，frontmatter 變更不會即時套用 |
| context 已壓縮要小心 | 若 subagent 曾觸發 auto-compact（>95% token），續跑時看到的歷史是壓縮後版本 |
| Plugin subagents 同樣可用 | 只要該 subagent 是這個 session 內 spawn 過的就有 agentId，不限定 built-in 還是自訂 |

### 何時不要用 SendMessage

- subagent 任務已完全結束、後續工作邏輯上獨立 → 直接開新 `Agent(...)` 比較乾淨
- 想要切換 model / 換 system prompt → 必須開新 `Agent(...)`，SendMessage 不能改設定
- 想刻意「重來一次」（前次方向錯了，希望忘掉舊脈絡）→ 開新 `Agent(...)`

---

## 5. 實測紀錄

於 2026-05-07 在 OpenStock 專案 session 內完成端到端驗證。

### 5.1 啟用前

- 環境變數未設、`.claude/settings.json` 也沒有 `env` 區塊
- `ToolSearch select:SendMessage` 回 `No matching deferred tools found`
- Agent 呼叫照常運作並回傳 agentId，但無法 SendMessage 喚醒

### 5.2 啟用過程的踩雷

第一次寫進 settings.json 時把 `env` 放進了 `permissions` 內部，**完全沒生效**：

```jsonc
// 錯誤
{ "permissions": { "allow": [...], "env": {...} } }

// 正確
{ "permissions": { "allow": [...] }, "env": {...} }
```

`env` 必須在**頂層**，不是 `permissions` 子物件。

### 5.3 啟用後（熱載入確認）

存檔修正 settings.json 的當下，系統 reminder 立刻通知 `SendMessage`、`TeamCreate`、`TeamDelete` 三個 deferred tools 上線 —— **不需要重啟 Claude Code**，settings.json 的 `env` 變更會被熱套用。

### 5.4 端到端 SendMessage 測試

```
Step 1: Agent(general-purpose, "記住祕密號碼：紫色蝙蝠 7891")
        → result: "已記住"
        → agentId: afdd92935d3ea2c72

Step 2: SendMessage(
          to="afdd92935d3ea2c72",
          summary="recall the secret number",
          message="請說出你剛才記住的祕密號碼")
        → {"success": true,
           "message": "Agent had no active task;
                       resumed from transcript in the background"}

Step 3: 系統推送 task-notification
        → <status>completed</status>
        → <result>紫色蝙蝠 7891</result>   ← 完整保留歷史
        → 15,825 tokens, 1,767 ms
```

### 5.5 與 schema 描述的關鍵差異

SendMessage 的 schema 描述明寫「Refer to teammates by name, never by UUID」，但 Agent 工具系統提示又說「use SendMessage with the agent's ID or name as the `to` field」。**實測：直接傳 agentId（UUID 形式）完全有效**，schema 描述是給 agent teams 那條路徑寫的。

繼續用 agentId 當 `to`，與本專案 orchestrator 流程相容。

---

## 6. 快速 Checklist

- [x] `.claude/settings.json` **頂層**加入 `env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`（已完成）
- [x] 用 `ToolSearch select:SendMessage` 驗證 schema 出得來（已驗證）
- [x] 端到端 SendMessage 續跑測試通過（已驗證 agentId `afdd92935d3ea2c72` 記住「紫色蝙蝠 7891」）
- [ ] 修改 [.claude/orchestrator.md](../.claude/orchestrator.md)：第二輪以後 executor 改用 SendMessage 續用
- [ ] 新增「executor_agent_id」欄位至 run log 模板（[wiki/runs/INDEX.md](../wiki/runs/INDEX.md) 對應檔）
