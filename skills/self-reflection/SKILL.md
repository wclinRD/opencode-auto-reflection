---
name: self-reflection
description: >
  自動反省機制。工具錯誤時即時分析原因、任務成功時提取可重用模式、
  session 結束時總結學習。根據學習內容自動產生新 skill 存入
  .agents/skills/ 或 .opencode/skills/，形成持續學習迴圈。
  支援三層洋蔥結構（表層緊急指令 / 中層標準流程 / 深層完整邏輯）。
compatibility: opencode
metadata:
  audience: all_agents
  workflow: meta_learning
  version: 1.0.0
  generated: manual
  status: active
---

## Layer 2: 可配置參數

所有參數均有內建預設值（Layer 1）。需調整時在 `skill()` 呼叫時傳入：

```yaml
# 使用預設值（不需額外設定）
# skill(name: "self-reflection")

# 自訂參數範例
# skill(name: "self-reflection", config: {
#   pattern_threshold: 5,
#   draft_max_per_session: 5
# })
```

| 參數 | 預設值 | 範圍 | 說明 |
|------|--------|------|------|
| `pattern_threshold` | 3 | 1-10 | L2 模式判定所需出現次數 |
| `analysis_threshold` | 2 | 1-5 | L1 分析觸發所需同類錯誤次數 |
| `archive_after_sessions` | 3 | 1-10 | draft 無匹配後自動歸檔的 session 數 |
| `bootstrap_session_count` | 3 | 1-10 | 引導期長度（期間降低閾值） |
| `draft_max_per_session` | 3 | 1-10 | 單一 session 最大 draft 產生數 |
| `log_retention_lines` | 500 | 100-2000 | 反省日誌保留行數 |
| `max_reflection_depth` | 2 | 1-5 | 最大反省嵌套深度（防止無限遞迴） |
| `health_check_interval` | 20 | 5-50 | 健康審查間隔（session 數） |
| `convergence_sessions` | 10 | 3-30 | 收斂判斷所需連續無新 skill session 數 |
| `review_threshold` | 5 | 3-10 | 無改善時標記 needs_review 的 session 數 |
| `log_path` | `~/.opencode/reflection-log.jsonl` | - | 反省日誌路徑 |
| `agent_skills_path` | `~/.agents/skills/` | - | Agent 通用 skill 存放路徑 |
| `opencode_skills_path` | `~/.opencode/skills/` | - | OpenCode 特定 skill 存放路徑 |
| `skill_prefix` | `auto-` | - | 自動產生 skill 的前綴 |
| `draft_status` | `draft` | draft/pending | 自動產生 skill 的初始 status |
| `log_level` | `minimal` | minimal/full | 日誌詳細程度 |
| `fast_mode_threshold` | 50 | 10-200 | 超過此 tool call 數切換快速模式 |

---

## Layer 1: 表層 — 緊急迷你指令（先讀這層）

當你時間有限或正在修復錯誤時，至少執行以下步驟：

### 工具報錯時（L1 最小行為）

```
1. 用 working_memory_add 記錄 {reflecting: true}（防止遞迴反省）
2. 檢查同類錯誤是否已在 log 出現 ≥ analysis_threshold 次
   - 首次 → 只 append log（1 行）
   - ≥ 2 次 → 跑完整分析 + 產生 draft skill
3. 用 working_memory_remove 清除 reflecting flag
4. 然後才開始修復錯誤（順序不可顛倒！）
```

### 任務完成時（L2 最小行為）

```
所有 todo 皆 completed → 快速掃描本 session 的 tool 序列
→ 若有 domain-specific 序列重複 ≥ pattern_threshold 次
→ 產生 draft skill
→ 否則跳過
```

### Session 結束 / 手動時（L3 最小行為）

```
tool_call_count ≥ 10 → 執行反省摘要輸出
→ 否則跳過（不需執行 L3）
```

---

## Layer 1: 中層 — 標準流程

### 執行前準備

每次 `skill(name: "self-reflection")` 載入時：
1. 讀取 `~/.opencode/reflection-log.jsonl`（不存在則建立空檔案）
2. 計算本 session 迄今 tool_call_count
3. 決定模式：`tool_call_count > fast_mode_threshold` 或 `context 使用率 > 70%` → 快速模式
4. 檢查 `working_memory` 中是否有 `reflecting: true` → 有則跳過全部反省（防止遞迴）

### L1: 工具報錯即時反省

**觸發條件**：工具回傳 error 或非預期結果

**Step 1 — 設定反射護盾**
```
working_memory_add({reflecting: true})
```

**Step 2 — 錯誤歸類 (F4)**
```
主 key: tool_name + error_code
副 key: operation_type（read/write/execute/parse/network）
無 error_code → 用 error_type（permission/network/parse/auth/unknown）
```

**Step 3 — 歸因檢查 (漏洞 51)**
問自己：這個錯誤是？
- (a) 工具本身的限制或 bug？
- (b) 我的判斷或參數錯誤？
- (c) 環境問題（網路/權限/檔案不存在）？

→ (b) 的情況：skill 應 focus 在改善判斷邏輯
→ (a)/(c) 的情況：skill 應 focus 在預防措施

**Step 4 — 節流判斷**
```
在 reflection-log.jsonl 中搜尋同類錯誤（主 key 相同）
count = 匹配行數 + 1

if count == 1:
  → 快速模式：只 append log（minimal 格式），跳過分析
  → 完整模式：append log + 標記 first_seen
if count >= analysis_threshold:
  → 執行 L1 完整分析（Step 5-7）
```

**Step 5 — 完整分析**
```
a. 確認錯誤訊息摘要
b. 確認操作上下文（當時在執行什麼 task）
c. 因果驗證 (F5)：這個教訓是否能預防原始錯誤？
   - 步驟 1: 識別錯誤發生的操作
   - 步驟 2: 確認 skill 教訓是否直接對應該操作
   - 步驟 3: 是 → confidence: high；否 → confidence: low
d. 決定是否產生 draft skill
```

**Step 6 — 產生或更新 draft skill**
```
if 同類錯誤已有 draft skill:
  → 更新既有 draft（加入新案例）
  → increment version
else:
  → 產生新 draft skill
  → 命名: {skill_prefix}{timestamp}-{error_type}-handler
  → 遵守品質模板（見深層）
```

**Step 7 — 更新日誌 + 清除護盾**
```
append log（full 格式）
working_memory_remove({reflecting: true})
```

### L2: 任務成功模式提取

**觸發條件**：所有 todo 皆為 completed（無 cancelled）

**Step 1 — 掃描 tool 序列**
回顧本 session 的工具呼叫序列，提取：
- 出現 ≥ pattern_threshold 次的 tool 組合
- 過濾掉通用序列 (F2)：read→edit, read→write, grep→read, glob→read

**Step 2 — Domain 模式記錄 (F3)**
對每個非通用序列，記錄三元組：
```
(goal_summary, approach, tool_sequence_hash)
```

**Step 3 — 跨 session 比對**
在 reflection-log.jsonl 中搜尋相同 `tool_sequence_hash`：
```
if match_count >= pattern_threshold:
  → 產生 draft skill（內容：該序列的標準操作流程）
if match_count < pattern_threshold && 仍在引導期（bootstrap_session_count 內）:
  → 降低閾值為 1，產生 draft skill
else:
  → 僅記錄到 log，不產生 skill
```

**Step 4 — Session 內 draft 上限**
```
if 本 session 已產生 draft_count >= draft_max_per_session:
  → 排隊到下個 session 處理（記錄到 log 的 pending 欄位）
```

### L3: 定期總結

**觸發條件**：
- 自動：`tool_call_count >= 10` 且 `no_pending_todos`
- 手動：user 呼叫 `/self-reflection` 或 `/self-reflection summary`

**執行內容**：
```
1. 合併重複 draft（相同 error pattern 只留一個，保留最高 version）
2. 清除低價值 draft (F1)：
   match_count == 0 && sessions_since_creation >= archive_after_sessions
   → status: archived
3. 動態頻率調整：
   連續 convergence_sessions 未產生新 skill
   → 反省頻率從「每次工具呼叫」降為「每 5 次工具呼叫」
   → 若又有新模式出現 → 頻率恢復
4. 收斂判斷 (F10)：
   檢查最近 convergence_sessions 的 log：
   - error_count == 0 → 真收斂（標記系統已穩定）
   - error_count > 0 但無新 skill → 反省失靈警告
5. Log rotation：
   保留最近 log_retention_lines 行
   舊行壓縮到 reflection-log.archive.jsonl
6. 輸出本 session 學習摘要（含 effectiveness）
```

---

## Layer 1: 深層 — 完整邏輯與邊界規則

### F1: 低價值判定標準

```
條件: match_count == 0 && sessions_since_creation >= archive_after_sessions
動作: frontmatter status → archived（不刪除檔案）
判斷者: L3 定期總結時自動執行
例外: confidence == high 的 draft 延長一倍時間
```

### F2: 通用序列黑名單

```
跳過以下序列（不視為可學習模式）:
- read → edit
- read → write
- grep → read
- glob → read
- read → read（連續讀取）
- bash → bash（連續執行）

僅當序列中包含 domain-specific 工具（如 git, npm, python, docker 等）
或序列長度 ≥ 3 且包含非 blacklisted 工具時才視為模式。
```

### F3: Domain 提取方法

```
記錄格式: {goal, approach, tool_sequence_hash, domain_tags}
- goal: 該任務的目標摘要（10 字內）
- approach: 解決方法摘要
- tool_sequence_hash: 工具名稱序列的 SHA256（不含參數）
- domain_tags: 從工具名稱推斷（如 git → version-control, npm → package-management）
不嘗試提取語意化的 domain 知識（避免模糊）。
```

### F4: 錯誤歸類粒度

```
一級分類 (主 key): tool_name + error_code
  error_code 優先順序:
  1. 系統 error code（ENOENT, EACCES, ETIMEOUT 等）
  2. HTTP status code（400, 401, 403, 404, 500 等）
  3. 工具特定 error code
  4. 無 code → error_type: permission/network/parse/auth/unknown/tool_specific

二級分類 (副 key): operation_type
  read / write / execute / parse / network / transform

比對規則: 主 key 相同 + 副 key 相同 → 同類錯誤
```

### F5: 因果驗證步驟

```
目標: 確認提議的 skill 教訓確實能預防原始錯誤

步驟:
1. 識別原始錯誤發生的精確操作（哪個工具 + 什麼參數）
2. 撰寫 skill 教訓（「使用 tool X 前應先做 Y」）
3. 反事實推理：如果遵循這個教訓，原始操作會成功嗎？
   - 會 → confidence: high
   - 不會 → confidence: low（教訓可能偏離 root cause）
   - 不確定 → confidence: medium

限制: 此驗證只執行 1 次。驗證失敗不遞迴。
```

### F6: 存放路由決策樹

```
學習內容涉及什麼？
├── system prompt / agent 行為模式 / 通用工具 API
│   └── → {agent_skills_path}<name>/SKILL.md
├── OpenCode plugin / extension / OpenCode 特定格式
│   └── → {opencode_skills_path}<name>/SKILL.md
├── 兼具兩者
│   └── → 先存 {agent_skills_path}，frontmatter 標記 also_applies_to: opencode
└── 無法判斷
    └── → 預設 {agent_skills_path}
```

### F7: User Feedback 擷取

```
正規表達式匹配（不區分大小寫）:
  正面反饋: /(對|好|正確|讚|完美|great|good|correct|perfect|nice)/
  負面反饋: /(不對|錯|不好|重做|wrong|incorrect|bad|redo|fix)/

行為:
  - 正面: increment skill confidence（上限 high）
  - 負面: 觸發該 skill 的重新評估
  - 無匹配: 不處理

記錄: reflection-log.jsonl 中新增 user_feedback 欄位
```

### F8: 健康審查標準

```
執行頻率: 每 health_check_interval 個 session 執行一次

stale 判定 (可歸檔):
  usage_count < 2 && sessions_alive >= health_check_interval

duplicate 判定 (可合併):
  sha256(file_content) == 已存在的檔案 → 刪除重複者
  name 不同但 description 相似度 > 80% → 合併內容

deprecated 判定 (應標註):
  工具/API 已不存在的 skill
  或 creation 後超過 30 天無任何匹配

overlap 判定 (應合併):
  兩個 skill 的 trigger_condition 有 50%+ 重疊 → 合併
```

### F9: 快速/完整模式切換

```
快速模式條件（任一成立）:
  1. 本 session tool_call_count > fast_mode_threshold
  2. context window 使用率 > 70%（依 agent 自我評估）
  3. user 明確要求「快一點」

快速模式行為:
  - L1: 只 append log（minimal 格式），跳過分析與 skill 產生
  - L2: 只記錄模式 hash，跳過比對與 skill 產生
  - L3: 只輸出摘要，跳過健康審查與 rotation

完整模式行為:
  全部正常執行
```

### F10: 收斂判斷

```
執行時機: L3 定期總結時

檢查範圍: 最近 convergence_sessions 個 session 的 log

判斷邏輯:
  if error_count_total == 0:
    → 真收斂。系統已穩定。可以降低反省頻率。
  if error_count_total > 0 && new_skills_created == 0:
    → 反省失靈！有錯誤發生但未產生任何 skill。
      可能原因: draft_max_per_session 已滿 / 所有錯誤都被 throttle 跳過
      解決: 強制執行一次完整 L1 分析（無論 throttle）
  if error_count_total > 0 && new_skills_created > 0:
    → 正常學習中。維持當前頻率。
```

### F11: L3 觸發條件

```
自動觸發:
  tool_call_count >= 10 && 所有 todo 皆 completed 或 cancelled
  （在 session 自然結束時執行）

手動觸發:
  user 輸入 /self-reflection → 完整 L3
  user 輸入 /self-reflection summary → 只輸出摘要
  user 輸入 /self-reflection review → 檢視 pending draft skills

禁止觸發:
  reflectionDepth >= max_reflection_depth（防止遞迴）
```

### F12: 洋蔥結構深度指引

```
執行原則:
  1. 先讀「表層」— 獲得最低可行指令
  2. 遇到「中層」提到但未詳述的概念時 → 查「深層」
  3. 所有 F1-F16 的詳細規則只在「深層」
  4. 不需預先讀完所有深層內容 — 需要時再查

緊急情況（錯誤修復中）:
  只執行表層指令，中層和深層等修復完成後再補讀
```

### F13: task_id 判斷

```
預設值: {project_name}_{YYYY-MM-DD}
  project_name: 從當前工作目錄名稱推斷

覆寫方式:
  user 可在 skill 載入時指定 task_id
  或透過 working_memory_add({task_id: "custom_name"}) 設定

跨 session 判斷:
  相同 project_name + 連續日期 → 視為同一 task
  超過 3 天無該 project 的活動 → 視為新 task
```

### F14: log_level 差別

```
minimal 格式（預設）:
  {"ts":"ISO8601","type":"error|pattern","tool":"name","error_code":"CODE","session":"id"}

full 格式:
  {"ts":"ISO8601","type":"error|pattern","tool":"name","error_code":"CODE",
   "operation":"context","resolution":"how_it_was_fixed","session":"id",
   "task_id":"name","platform":"darwin|linux","agent_type":"smart|plan|architect",
   "confidence":"high|medium|low","user_feedback":"positive|negative|null"}
```

### F15: Effectiveness 計算

```
計算時機: 每次 L3 總結時，對所有 active skill 計算

公式:
  baseline_error_rate = 該 skill 產生前 review_threshold 個 session 的同類錯誤次數 / review_threshold
  current_error_rate = 該 skill 產生後 review_threshold 個 session 的同類錯誤次數 / review_threshold
  improvement = (baseline_error_rate - current_error_rate) / baseline_error_rate

判定:
  improvement >= 0.2 → effective（維持 active）
  0 < improvement < 0.2 → needs_review（可能無效）
  improvement <= 0 → ineffective（自動降級為 draft）

注意: baseline_error_rate == 0 時跳過計算（無從比較）
```

### F16: L1 節流 scope

```
「完整分析」包含:
  ① 記錄錯誤到 log（full 格式）
  ② 錯誤歸類（主 key + 副 key）
  ③ 比對歷史 log 中的同類錯誤
  ④ 歸因檢查（工具問題 vs 判斷錯誤）
  ⑤ 因果驗證
  ⑥ 產生或更新 draft skill
  ⑦ 記錄 effectiveness baseline

「只計數」包含:
  ① 記錄錯誤到 log（minimal 格式）
  ② 錯誤歸類（主 key + 副 key）
  ③ increment 同類錯誤計數器
  （跳過 ④-⑦，因已由首次分析覆蓋）
```

---

## 存放路由決策樹（實作）

當需要產生新 skill 時，依以下順序判斷存放位置：

```
1. 學習內容是否涉及 system prompt / agent 行為 / 通用工具 API？
   是 → {agent_skills_path}
2. 學習內容是否涉及 OpenCode plugin / extension / 特定格式？
   是 → {opencode_skills_path}
3. 兼具兩者？
   先存 {agent_skills_path}，frontmatter 標記 also_applies_to: opencode
4. 無法判斷？
   預設 {agent_skills_path}
```

產生 skill 前的防護：
```
1. 掃描目標目錄確認無同名衝突
   有衝突 → 名稱加後綴 -v2, -v3...
2. 掃描 deleted_skills 清單（從 reflection-log.jsonl 中讀取）
   已 deleted → 跳過產生
3. Atomic write: 寫入 .tmp → rename → 完成
4. 寫入後立即驗證格式（必要 frontmatter 欄位）
```

---

## 品質控制

### 自動產生 skill 的 frontmatter 模板

```yaml
---
name: {skill_prefix}{timestamp}-{error_type}-handler
description: >
  自動產生的 skill。{一句話概述解決什麼問題}
compatibility: opencode
metadata:
  generated: auto
  status: {draft_status}
  version: 1
  parent: self-reflection
  source_session: {session_id}
  confidence: {high|medium|low}
  platform: {darwin|linux|all}
  also_applies_to: {null|opencode}
  original_error: {error_code}
  trigger_condition: {何時觸發}
---
```

### 內容品質模板

自動產生的 skill 必須包含以下三節：
```
## 觸發條件（何時使用這個 skill）
精確描述觸發情境

## 解決步驟（逐步操作）
1. ...
2. ...
3. ...

## 驗證（如何確認解決成功）
確認步驟
```

缺乏任一節 → 重新生成。

### 防止循環繁衍（漏洞 35/46）

```
規則:
1. 所有 self-reflection 產生的 skill 標記 parent: self-reflection
2. 如果 child skill 執行時出錯，更新 child 本身（不產生新的 meta-skill）
3. reflectionDepth >= max_reflection_depth 時跳過所有 L1/L2 分析
4. depth 計算包含隱式反省（在反省過程中又呼叫反省工具）
```

---

## 與強制循環演算法整合

系統的強制循環演算法（每個工具呼叫後檢查 todo）擴充為：

```
原始循環:
  工具呼叫 → 查 todo → 有 pending? → 執行 → 回到查 todo

擴充循環:
  工具呼叫
    → reflectionDepth < max_reflection_depth?
       是 → 檢查上個工具結果
            → 有錯誤？ → L1 反省（設定 reflecting flag）
            → 無錯誤？ → 繼續
       否 → 跳過反省
    → 查 todo → 有 pending? → 執行
    → 所有 completed? → L2 模式提取
    → session 結束? → L3 定期總結
    → 回到工具呼叫
```

## 與 Smart Heartbeat 整合

當 Heartbeat 注入續行提示時：
```
1. 檢查 working_memory 是否有 reflecting: true
   → 有：等待反省完成再繼續執行任務
   → 無：正常執行續行
2. reflection flag 在 working_memory 中跨續行存活
3. 續行後不重置 reflectionDepth
```

---

## 安全與隱私

1. **不記錄工具參數內容** — log 只記錄 tool_name + error_code，不記錄路徑/檔名/資料
2. **platform 標記** — 所有 skill 標記適用平台，避免跨平台誤用
3. **deleted_skills 清單** — 記錄在 `reflection-log.jsonl` 中，type 為 `deletion`：
   ```
   {"ts":"ISO8601","type":"deletion","tool":"skill_name","session":"id"}
   ```
   產生新 skill 前掃描此清單，已 deleted 的不再生產。
4. **log_level 控制** — minimal 模式完全不記錄操作上下文
5. **回滾方式** — 刪除 `~/.agents/skills/self-reflection/` + `reflection-log.jsonl` 即可完全移除

---

## 首次使用引導（漏洞 42）

這是你第一次使用 self-reflection skill。預期行為：

1. **第 1-3 session（引導期）**：
   - 模式閾值從 3 降為 1（更容易產生 draft）
   - 可能還沒有足夠資料產生有意義的 skill
   - 每次 L3 會輸出學習進度

2. **第 4+ session（正常期）**：
   - 恢復正常閾值
   - 跨 session 比對開始產出有價值的 skill
   - 錯誤模式開始被系統性覆蓋

3. **長期（20+ session）**：
   - 常見錯誤已被 skill 覆蓋
   - 反省頻率自動降低（收斂）
   - 系統進入穩定維護階段

