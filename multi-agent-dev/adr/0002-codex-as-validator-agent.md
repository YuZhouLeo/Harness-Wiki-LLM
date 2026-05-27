# ADR-0002: 採用 Codex 作為驗證代理

- 狀態：Accepted
- 日期：2026-05-06

## 脈絡

[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) 將開發流程拆成「規劃 → 驗證 → 執行 → 經驗」四個角色，其中**驗證**是整個工作流的品質守門關鍵。需要決定：誰來扮演驗證者？

候選方案的根本問題是「**同源誤差（self-review bias）**」：

- Claude 既規劃又驗證 → 規劃時有的盲點，驗證時還是看不到
- 同一模型家族的 prompt 變體 → 仍共享同樣的訓練分佈、同樣的審美偏好
- 沒有跨模型對抗 → 「自己審自己」變裝成「驗證」，但本質是橡皮圖章

OpenStock 之前以單一 Claude 主對話「邊規劃邊改實作」，已經出現過數次「我規劃時的判斷沒被質疑就直接被實作」的案例（Harness 計劃導入前的混亂期）。要打破這個迴圈，**驗證者必須是不同模型**——不只是不同 prompt，是**不同公司、不同訓練資料、不同 alignment 取向**的模型。

OpenAI 推出的 [codex-plugin-cc](https://github.com/openai/codex-plugin-cc) 把 Codex 包成 Claude Code 的子命令（`/codex:review`、`/codex:adversarial-review`），可以直接從 Claude Code 內部調用。這是實務上跨模型驗證最低 friction 的整合路徑。

## 決策

採用 **Codex（透過 codex-plugin-cc）** 作為多代理工作流的驗證代理。

具體規則：

1. **整合方式**：以 `/codex:review` 與 `/codex:adversarial-review` slash commands 在主代理調度時觸發；不在 `.claude/agents/` 建立 subagent 檔（驗證者不是 Claude subagent）
2. **session 管理**：`--background` 啟新 session 取得 session-id；後續輪次 `--resume <session-id>` 接續記憶。詳見 [plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §3.1
3. **角色拆分**：驗證代理拆為「計劃顧問（諮詢）」與「實作驗證（守門）」兩個邏輯角色，但共用 Codex 後端，僅 prompt template / session 分離。詳見 [ADR-0003](0003-split-plan-advisor-and-impl-validator.md)
4. **認證方案**：Phase 1 起步用 ChatGPT 訂閱；Phase C 評估後再決定是否切 `OPENAI_API_KEY`（[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §9 #5）
5. **stop hook**：`/codex:setup` 階段選擇**關閉自動 stop hook**，避免每個 Claude 回應結束都被自動 review，干擾調度節奏

評估過的替代：

- **Claude self-review**（同 Claude 換個 prompt 自審）：成本最低，但同源誤差問題完全沒解。**否決**——這正是要避免的反模式
- **改用其他 OpenAI 模型（GPT-4 / o1）但不透過 codex-plugin-cc**：自己寫 API 包裝層，session 管理、認證、prompt 注入都得從零做。**否決**——重複造輪子，且失去 plugin 提供的 `--background` / `--resume` 機制
- **不要驗證層，直接信任 Claude self-review**：放棄品質守門。**否決**——已知會回到工作流導入前的混亂狀態
- **三模型輪審（Claude + Codex + Gemini）**：理論上更穩，但 token 成本 ×3、調度邏輯複雜度暴增、session 管理 9 倍狀態。**否決**——Phase 1 不需要這個層級的保險

## 後果

**好處**

- 跨模型驗證消除同源誤差，規劃 / 實作的盲點有不同視角質疑
- codex-plugin-cc 提供開箱即用的 session 管理（`--background` / `--resume`），主代理不需要自己維護對話狀態
- 主代理本身保持「純調度」角色，不做技術判斷——驗證職責清楚劃分

**壞處**

- **多一個外部依賴**：需要 ChatGPT 訂閱或 OpenAI API key；Codex 服務 down 時整個工作流卡住（緩解：實作驗證 3 輪 FAIL 自動產出 handoff MD，[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §3.2.2）
- **session 狀態管理複雜度上升**：一次工作流會有 advisor-session-id + impl-session-id 兩個獨立 session，主代理必須記住哪個是哪個、不可互相 resume（緩解：[ADR-0003](0003-split-plan-advisor-and-impl-validator.md) 風險與 [plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §4.2.4 共通禁止項）
- **token 成本提升**：Claude 的 planner/executor/lesson-keeper 之外又加了 Codex 兩個 session，總 token 消耗比單代理工作流明顯增加（緩解：Phase 2 量化監控；[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §7 風險表第 1 條）

**需要持續觀察**

- Codex session-id 跨主對話會話的失效頻率：Phase 2 觀察 5 個任務的失效率，若 ≥ 30% 需設計 fallback（[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §9 #2）
- Codex 假陽性率：若實作驗證頻繁卡死但實作其實沒問題，需調整對抗 prompt 嚴格度（[plans/多代理協作工作流計劃.md](C:/Users/yuzho/.claude/skills/multi-agent-dev/spec/多代理協作工作流計劃.md) §7 風險表第 2 條）
- Codex 與 Claude 評審準則的長期一致性：若兩者對「合格」的定義差太大，工作流會在驗證階段持續卡關
