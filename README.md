# opencode-auto-reflection

讓 OpenCode agent 擁有自動學習反思能力。工具報錯時分析原因、任務成功時提取模式、自動產生新 skill 形成持續學習迴圈。

## 快速安裝

### 一鍵安裝

```bash
# 1. 複製 skill 到 agents 目錄（所有 agent 通用）
cp -r skills/self-reflection ~/.agents/skills/

# 2. 初始化反省日誌
cp reflection-log.jsonl ~/.opencode/
```

### 手動安裝

```bash
# 1. 建立 skill 目錄
mkdir -p ~/.agents/skills/self-reflection/

# 2. 複製 SKILL.md
cp skills/self-reflection/SKILL.md ~/.agents/skills/self-reflection/

# 3. 初始化反省日誌（不存在時自動建立）
touch ~/.opencode/reflection-log.jsonl
```

### 驗證安裝

```bash
ls ~/.agents/skills/self-reflection/SKILL.md     # 應存在
ls ~/.opencode/reflection-log.jsonl               # 應存在
```

## 使用方式

### 載入 Skill

在 OpenCode TUI 中輸入：

```
/self-reflection
```

或者在其他 skill 或 CLAUDE.md 中自動載入：

```yaml
# CLAUDE.md 範例
on_start:
  - skill(name: "self-reflection")
```

### 子指令

| 指令 | 功能 |
|------|------|
| `/self-reflection` | 完整反省 -- 執行 L1+L2+L3 |
| `/self-reflection summary` | 只看本 session 學習摘要 |
| `/self-reflection review` | 檢視 pending draft skills，決定是否發布 |

### 設定參數

可在 skill 載入時覆寫預設參數：

```yaml
skill(name: "self-reflection", config: {
  pattern_threshold: 5,        # 模式判定所需次數（預設 3）
  draft_max_per_session: 5,    # 單 session 最大 draft 數（預設 3）
  log_level: "full"            # 日誌詳細程度（預設 minimal）
})
```

完整參數表請見 SKILL.md 的「Layer 2: 可配置參數」章節。

## 運作原理

### 三層反省架構

```
L1 即時反省 —— 工具報錯時
  ├── 首次錯誤：只記錄到日誌（節流）
  ├── 同類錯誤 ≥ 2 次：完整分析 root cause
  └── 自動產生 draft skill

L2 模式提取 —— 任務成功完成時
  ├── 掃描本次 tool 使用序列
  ├── 過濾通用序列（read→edit 等）
  ├── 跨 session 比對相同序列
  └── 達到閾值時產生 draft skill

L3 定期總結 —— Session 結束 / 手動觸發
  ├── 合併重複 draft
  ├── 清除低價值項目
  ├── 動態調整反省頻率
  └── 輸出學習摘要
```

### 自動 Skill 產生

當同一錯誤模式出現多次，或發現跨 session 可重用的成功模式時，self-reflection 會自動：

1. 分析錯誤原因（歸因檢查 — 是工具問題還是 agent 判斷錯誤）
2. 產生結構化的 SKILL.md（含 frontmatter + 觸發條件 + 解決步驟 + 驗證）
3. 根據內容自動路由到正確位置：
   - Agent 通用行為 → `~/.agents/skills/`
   - OpenCode 特定行為 → `~/.opencode/skills/`
4. 標記為 `draft` 狀態，經使用者確認後升為 `active`

### 防護機制

| 機制 | 說明 |
|------|------|
| Reflection Guard | 防止無限遞迴反省（depth ≥ 2 停止） |
| 節流 | 同類錯誤首次只記錄，二次才分析 |
| 品質管控 | 自動產生的 skill 必須通過格式驗證 |
| 隱私保護 | 日誌只記錄 error_code，不記錄參數內容 |
| 動態頻率 | 連續 10 session 無新學習 → 反省頻率自動降低 |
| 來源標記 | 所有自動 skill 標記 `generated: auto`，可追溯來源 |

## 檔案結構

```
~/.agents/skills/self-reflection/
└── SKILL.md              # 核心反省技能

~/.opencode/
└── reflection-log.jsonl   # 反省日誌（JSONL 格式，append-only）
```

## 與其他工具整合

### Smart Heartbeat Plugin

如使用 Smart Heartbeat Plugin，self-reflection 會自動在續行時保留反省狀態，避免續行打斷反省流程。

### 強制循環演算法

self-reflection 已與 OpenCode 的強制循環演算法整合，在每次工具呼叫後自動檢查是否需要反省。

## 架構設計

完整的設計審查文件請見 [docs/design-review.md](docs/design-review.md)。

設計歷經 **8 輪工程審查**，共修復 **96 項漏洞**：

| 輪次 | 主題 | 漏洞數 |
|------|------|--------|
| 第 1 輪 | 架構設計 | 8 |
| 第 2 輪 | 實作細節 | 9 |
| 第 3 輪 | 邊界情況 | 10 |
| 第 4 輪 | 系統整合 | 8 |
| 第 5 輪 | 深層假設 | 5 |
| 第 6 輪 | UX/演化 | 10 |
| 第 7 輪 | 根本假設 | 10 |
| 第 8 輪 | 硬編碼/模糊空間 | 31 |
| 計畫審查 | 執行漏洞 | 5 |

## 從原始碼安裝

```bash
git clone https://github.com/wclinRD/opencode-auto-reflection.git
cd opencode-auto-reflection

# Agent 通用 skill（所有 agent 可用）
cp -r skills/self-reflection ~/.agents/skills/

# OpenCode 專用 skill（可選）
cp -r skills/self-reflection ~/.opencode/skills/

# 反省日誌
cp reflection-log.jsonl ~/.opencode/
```

## 解除安裝

```bash
rm -rf ~/.agents/skills/self-reflection/
rm -f ~/.opencode/reflection-log.jsonl
```

## License

MIT
