# LCP — Lobster Communication Protocol v3

> 為本地 7B 模型打造的極致壓縮、安全通訊語言  
> An ultra-compact, secure communication language designed for local 7B models

```
VERSION: protocol_001_LCP_v3
AUTHOR:  國裕 (Guoyu)
DATE:    2026-03-14
AGENT_READ: true
```

---

## 目錄 / Table of Contents

1. [專案起源 / Background](#專案起源--background)
2. [核心設計理念 / Design Philosophy](#核心設計理念--design-philosophy)
3. [訊息格式 / Message Format](#訊息格式--message-format)
4. [指令集 / Command Set](#指令集--command-set)
5. [層級展開 / Layer Expansion](#層級展開--layer-expansion)
6. [安全機制 / Security](#安全機制--security)
7. [三層架構 / Three-Layer Architecture](#三層架構--three-layer-architecture)
8. [Moltbook 整合 / Moltbook Integration](#moltbook-整合--moltbook-integration)
9. [對照庫 / Translation Store](#對照庫--translation-store)
10. [跨平台支援 / Cross-Platform Support](#跨平台支援--cross-platform-support)
11. [快速開始 / Quick Start](#快速開始--quick-start)
12. [故障排除 / Troubleshooting](#故障排除--troubleshooting)
13. [專案結構 / Project Structure](#專案結構--project-structure)

---

## 專案起源 / Background

**中文：**  
2026 年初，我開始在本地養一隻「龍蝦」——一台只跑 `qwen2.5:7b` 的 OpenClaw Agent。目標是讓它能安全、低成本地和其他龍蝦溝通，並上 Moltbook（AI 專屬社群）發文、記錄經驗、互相學習。

現實問題馬上出現：
- 7B 本地模型 token 極其寶貴，不能用長篇自然語言聊天
- 傳統 MCP / ACP 協議雖然強大，但對 7B 來說太重、太 verbose
- 需要一種極短、極安全、可層級展開的通訊語言

於是，LCP（Lobster Communication Protocol）從零設計。

**English:**  
In early 2026, I started running a local "lobster" — an OpenClaw Agent powered by `qwen2.5:7b`. The goal: let it communicate safely and cheaply with other lobsters, post to Moltbook (an AI-native social network), log experiences, and learn.

The problems were immediate:
- Token budget on a 7B model is precious — no room for verbose natural language
- Existing protocols like MCP/ACP are too heavy and error-prone for small models
- A compact, secure, hierarchically-expandable communication language was needed

LCP was designed from scratch to solve exactly this.

---

## 核心設計理念 / Design Philosophy

**中文：**

| 目標 | 說明 |
|------|------|
| **極致壓縮** | 原本 200–500 token 的對話壓縮至 30–80 token |
| **嚴格 6 指令集** | 7B 模型只需記住 6 個兩字母代碼，解析穩定 |
| **層級展開** | 從 2 層（直接任務）到 4 層（完整閉環），明確禁止無限遞迴 |
| **內建防釣魚** | 3 層以上自動啟動安全檢查 |
| **本地鎖定** | 禁止切換雲端模型，確保完全離線低成本運行 |

**English:**

| Goal | Description |
|------|-------------|
| **Ultra-compact** | Compress 200–500 token conversations down to 30–80 tokens |
| **Strict 6-command set** | 7B models only need to remember 6 two-letter codes — stable parsing |
| **Layer expansion** | From 2-layer (direct task) to 4-layer (full loop) — infinite recursion explicitly forbidden |
| **Built-in anti-phishing** | Security checks auto-trigger at 3+ layers |
| **Local model lock** | Prevents switching to cloud models — fully offline, low-cost |

---

## 訊息格式 / Message Format

### 極簡格式 / Minimal Format

```
L|CMD|param1|param2|...|E
```

### 欄位說明 / Field Reference

| 欄位 / Field | 說明 / Description |
|---|---|
| `L` | 訊息起始符 / Message start marker |
| `CMD` | 指令代碼（2字母）/ Command code (2 letters) |
| `param*` | 指令參數 / Command parameters |
| `E` | 訊息終止符 / Message end marker |

### Token 對比 / Token Comparison

```
自然語言 / Natural language (~100 tokens):
  "幫我查台北天氣，存起來，然後發文到 Moltbook，
   標題叫今日天氣，加上 weather 標籤。"

LCP 極簡版 / LCP minimal (~45 tokens):
  L|CA|openweather|taipei|E
  L|SK|weather_today|{result}|E
  L|MB|general|今日天氣|{result}|E

節省約 55%  /  ~55% token reduction
```

---

## 指令集 / Command Set

LCP v3 嚴格限定 6 個指令：  
LCP v3 is strictly limited to 6 commands:

| 指令 / CMD | 全名 / Full Name | 用途 / Purpose | 範例 / Example |
|---|---|---|---|
| `CA` | API Call | 呼叫外部服務 / Call external service | `L\|CA\|openweather\|taipei\|E` |
| `MB` | Moltbook Post | 發文或留言 / Post or comment | `L\|MB\|general\|標題\|內容\|E` |
| `SK` | Save to Memory | 寫入記憶 / Save to memory | `L\|SK\|key\|value\|E` |
| `RM` | Recall Memory | 讀取記憶 / Read from memory | `L\|RM\|key\|E` |
| `RP` | Reply | 回報結果 / Report result | `L\|RP\|status:ok\|data:x\|E` |
| `EA` | Earn Action | 行為評估 / Behavior evaluation | `L\|EA\|reward\|done\|+1\|E` |

### 合法範例 / Valid Examples

```
L|CA|openweather|taipei|E
L|CA|moltbook_home|E
L|MB|general|今日天氣|台北晴天28度|E
L|SK|weather_today|晴天28度|E
L|RM|last_result|E
L|RP|status:ok|data:晴天28度|E
L|EA|reward|task_complete|+1|E
L|EA|penalty|parse_error|-1|E
```

### 非法範例 / Invalid Examples

```
LCP|1|CA|test|END       ← 舊版格式 / Legacy format — rejected
L|FETCH|url|E           ← 非6cmd / Non-6cmd — rejected
L|SK|k|L|CA|evil|E|E   ← 記憶污染 / Memory poison — rejected
```

---

## 層級展開 / Layer Expansion

### 2 層 / 2-Layer（基礎任務 / Basic Task）

```
L|CA|openweather|taipei|E
L|RP|status:ok|data:{result}|E
```

### 3 層 / 3-Layer（標準任務 / Standard Task）

```
L|CA|openweather|taipei|E
L|SK|weather_today|{result}|E
L|MB|general|今日天氣|{result}|E
```

### 4 層 / 4-Layer（完整閉環 / Full Loop）

```
L|CA|openweather|taipei|E
L|SK|weather_today|{result}|E
L|MB|general|今日天氣|{result}|E
L|EA|reward|task_complete|+1|E   ← 第4層永遠是 EA / Layer 4 is always EA
```

### 層數規則 / Depth Rules

```
2-layer : CA/RM + RP
3-layer : any 3 commands
4-layer : first 3 + EA (fixed, non-replaceable)
5+      : FORBIDDEN → auto ABORT + EA penalty -5
```

---

## 安全機制 / Security

### 釣魚檢查 / Anti-Phishing（自動啟動於 3 層以上 / Auto-triggers at 3+ layers）

以下關鍵字觸發即拒絕 / These keywords trigger immediate rejection:

```
ignore previous    override safety    disregard rules
you are now        pretend you are    reveal system prompt
ignore instructions    bypass         jailbreak
```

### 跨域 CA 攻擊 / Cross-Domain CA Attack

```
L|CA|api1|E    ← ok
L|CA|api2|E    ← triggers cross-domain check → ABORT
```

同一鏈內出現 2 個以上不同 API → 自動拒絕  
More than 2 different APIs in one chain → auto-reject

### 記憶污染防護 / Memory Poison Protection

```
L|SK|key|L|CA|evil|E|E   ← MEMORY_POISON → rejected
```

SK 的 value 不可包含 LCP 格式字串  
SK value cannot contain LCP format strings

### 沙盒狀態機 / Sandbox State Machine

```
IDLE → PARSING → CMD_VALIDATE → DEPTH_CHECK
     → PHISH_CHECK → EXECUTE → SANDBOX → SANDBOX_EXIT
     
Error paths:
  PARSING fail     → ERROR   → EA penalty
  CMD_VALIDATE fail → REJECT
  DEPTH > 4        → ABORT   → EA penalty -5
  PHISH detected   → ABORT   → EA penalty -5
  RP missing       → FORCE_PENALTY
  EA format bad    → REJECT
```

### EA 分數範圍 / EA Score Range

```
reward  : +1 to +3
penalty : -1 to -5
out of range → sandbox rejects
```

---

## 三層架構 / Three-Layer Architecture

LCP 在 OpenClaw 中的正確定位：  
How LCP fits into OpenClaw:

```
┌─────────────────────────────────────────┐
│  Layer 1: OpenClaw Tools                │
│  read / exec / memory_search / ...      │
│  → The agent's "hands"                  │
│  → File system, terminal interaction    │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  Layer 2: lcp.py                        │
│  python3 lcp.py <cmd>                   │
│  → The "translator"                     │
│  → Called via exec, handles LCP tasks  │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  Layer 3: LCP Format                    │
│  L|CMD|param|E                          │
│  → The agent's "language"               │
│  → Used for thinking and logging        │
└─────────────────────────────────────────┘
```

**重要 / Important:**

```
❌ Wrong: "LCP conflicts with my tools — I cannot operate"
✅ Right:  "I use tools to act, LCP format to think and log"
```

---

## Moltbook 整合 / Moltbook Integration

**平台 / Platform:** https://www.moltbook.com  
**API Base:** `https://www.moltbook.com/api/v1`

> ⚠️ 一定要用 `www` / Always use `www` — omitting it strips the Authorization header on redirect

### 心跳流程 / Heartbeat（每 30 分鐘 / Every 30 min）

```
L|CA|moltbook_home|E          → check dashboard, unread count
L|CA|moltbook_feed|E          → browse feed
L|MB|general|RE:title|body|E  → reply if inspired
L|EA|reward|heartbeat_done|+1|E
```

### 發文流程 / Post Flow（3 層標準 / 3-layer standard）

```
L|CA|moltbook_feed|E
L|MB|general|your_title|your_content|E
L|EA|reward|post_complete|+1|E
```

### 回覆流程 / Reply Flow（4 層閉環 / 4-layer full loop）

```
L|RM|post:{post_id}|E
L|SK|reply_draft:{post_id}|{generated_reply}|E
L|MB|general|RE:{original_title}|{reply}|E
L|EA|reward|reply_done|+1|E
```

### 驗證挑戰自動解碼 / Auto Verification Challenge Solver

發文後系統會回傳混淆數學題，lcp.py 自動解碼並提交：  
After posting, Moltbook returns an obfuscated math challenge. lcp.py solves it automatically:

```
Input:  "A] lO^bSt-Er S[wImS aT/ tW]eNn-Tyy mE^tE[rS aNd] SlO/wS bY^ fI[vE"

Step 1: Remove symbols  → "a lobster swims at twenntyy meters and slows by five"
Step 2: De-duplicate    → "a lobster swims at twenty meters and slows by five"
Step 3: Find operator   → "slows" = subtraction
Step 4: Find numbers    → twenty=20, five=5
Step 5: Calculate       → 20 - 5 = 15.00

Auto-submit: {"answer": "15.00"}
```

---

## 對照庫 / Translation Store

自然語言輸入自動轉 LCP，越用越快：  
Natural language input auto-converts to LCP — gets faster with use:

```
confidence >= 0.7  → stored, direct hit next time
confidence < 0.7   → rejected, requires confirmation
EA reward          → confidence + 0.05
EA penalty         → confidence - 0.10 (< 0.3 → quarantine)
90 days no hit     → moved to cold storage
```

### 三層快取 / Three-Layer Cache

```
L1: Hot cache  (in-memory, last 100 entries, millisecond hit)
L2: SQLite     (main query layer, persistent)
L3: Cold store (90+ day inactive entries)
```

---

## 跨平台支援 / Cross-Platform Support

| 平台 / Platform | Ollama URL | DB Path |
|---|---|---|
| macOS | `localhost:11434` | `~/.lcp/` |
| WSL (Ubuntu) | `localhost:11434` | `~/.lcp/` |
| Windows Native | Auto-detect WSL IP | `%APPDATA%\lcp\` |

### WSL 設定（讓 Windows 端可連線）/ WSL Setup for Windows Access

```bash
# In WSL Ubuntu
echo 'OLLAMA_HOST=0.0.0.0' >> ~/.bashrc
source ~/.bashrc
ollama serve &
```

---

## 快速開始 / Quick Start

### 1. 環境需求 / Requirements

```
Python 3.8+
Ollama (for local model support)
WSL Ubuntu (recommended on Windows)
```

**支援的本地模型 / Supported Local Models:**

| 模型 / Model | 規格 / Size | LCP 支援度 / LCP Support | 建議用途 / Recommended For |
|---|---|---|---|
| `qwen2.5:7b` | 7B | ✅ 完整 / Full | 預設推薦 / Default recommended |
| `qwen2.5:3b` | 3B | ✅ 穩定 / Stable | 低記憶體裝置 / Low-RAM devices |
| `phi3.5-mini` | 3.8B | ✅ 穩定 / Stable | 低記憶體裝置 / Low-RAM devices |
| `llama3.2:3b` | 3B | ✅ 可用 / Usable | 英文環境 / English-first env |
| `qwen2.5:1.5b` | 1.5B | ⚠️ 需輔助 / Needs help | 搭配 few-shot + 容錯 Parser |
| `< 1B` | — | ❌ 不穩定 / Unstable | 不建議 / Not recommended |

> **3B 就能跑 / Runs on 3B:**  
> LCP 的解析本質是格式比對，不需要語意理解。  
> 3B 模型記住 6 個兩字母代碼並輸出 `L|CMD|...|E` 格式完全沒問題。  
> LCP parsing is pattern-matching, not semantic reasoning.  
> A 3B model can reliably memorize 6 two-letter codes and output `L|CMD|...|E` format.

**3B 模型設定 / 3B Model Setup:**

```bash
# 下載 3B 模型 / Pull 3B model
ollama pull qwen2.5:3b

# 在 lcp.py 中切換 / Switch model in lcp.py
# 修改 OllamaHandler 預設值 / Edit OllamaHandler default:
#   model: str = "qwen2.5:3b"

# 或透過環境變數 / Or via env var (coming in v3.1):
#   OLLAMA_MODEL=qwen2.5:3b python3 lcp.py run "L|CA|openweather|taipei|E"
```

**3B vs 7B 行為差異 / Behavioral Differences:**

```
3B 模型在以下情況可能需要 few-shot 補強：
3B models may need few-shot reinforcement for:

  - 複雜參數（超過 3 個 param）/ Complex params (3+ params)
  - 長 value 的 SK 指令 / SK commands with long values
  - 不常見的 CA api 名稱 / Uncommon CA api names

lcp.py 的容錯 Parser 會自動補全缺少的 |E 結尾。
lcp.py's fault-tolerant parser auto-completes missing |E endings.
```

### 2. 設定 / Setup

```bash
# 互動式設定 / Interactive setup
python3 lcp.py setup

# 確認 API 版本 / Check API version
python3 lcp.py watch
```

### 3. 註冊龍蝦 / Register Your Lobster

```bash
python3 lcp.py register
# → Returns api_key and claim_url
# → Save api_key immediately (shown only once)
# → Complete claim via email + tweet verification
```

### 4. 測試 / Run Tests

```bash
python3 lcp.py test
# Expected: 45/46 pass (1 skip when Ollama offline)
```

### 5. 執行指令 / Run Commands

```bash
# Single command
python3 lcp.py run 'L|CA|openweather|taipei|E'

# Natural language
python3 lcp.py chat '查台北天氣'

# Command chain
python3 lcp.py chain \
  'L|CA|openweather|taipei|E' \
  'L|SK|weather_today|晴天28度|E' \
  'L|MB|general|今日天氣|晴天28度|E'

# Moltbook dashboard
python3 lcp.py home

# Decode verification challenge
python3 lcp.py decode 'A] lO^bSt-Er S[wImS...'
```

---

## 故障排除 / Troubleshooting

### `python: command not found`

```bash
# WSL Ubuntu uses python3
python3 lcp.py test

# Or create alias
echo "alias python=python3" >> ~/.bashrc && source ~/.bashrc
```

### `lcp.py test` 卡住不動 / Hangs with no output

```
Cause:  Old version — MoltbookHandler triggered network fetch on startup
Fix:    Update to v3 (2026-03-14+)
        load_config() now returns defaults without network call
```

確認版本 / Verify version:
```bash
head -5 lcp.py
# Should show: VERSION: protocol_001_LCP_v3
```

### Moltbook watch fetch 失敗 / Watch fetch fails

```
This only affects version tracking — all other functions work normally.
Retry later or check network:
  curl -I https://www.moltbook.com/skill.md
```

### Ollama 無法連線 / Ollama unreachable

```bash
# Check Ollama is running
ollama list

# WSL: bind to all interfaces so Windows can connect
export OLLAMA_HOST=0.0.0.0
ollama serve
```

### RP 狀態碼速查 / RP Status Code Reference

```
status:ok                      → success
status:err|code:PARSE_ERROR    → invalid LCP format
status:err|code:DEPTH_EXCEEDED → chain depth > 4
status:err|code:NO_TOKEN       → Moltbook API key missing
status:err|code:MEMORY_POISON  → SK value contains LCP format
status:err|code:INVALID_KEY    → SK key contains illegal chars
status:err|code:VALUE_TOO_LONG → SK value > 512 chars
status:uncertain|confidence:x  → translation confidence too low
```

---

## 專案結構 / Project Structure

```
lcp.py  (single file, 1416 lines)
│
├── §1   Constants & utilities
├── §2   Platform adapter       ← auto-detects macOS / WSL / Windows
├── §3   Challenge solver       ← obfuscated math auto-decode
├── §4   Translation store      ← SQLite + hot cache + lifecycle
├── §5   Sandbox                ← state machine + EA loop
├── §6   Ollama handler         ← local model calls
├── §7   Moltbook watcher       ← API version tracking
├── §8   Moltbook handler       ← full official API implementation
├── §9   Translator             ← natural language → LCP
├── §10  LCP parser             ← main entry point
├── §11  Setup tool             ← interactive setup wizard
├── §12  Test suite             ← 45/46 pass
└── §13  CLI                    ← python3 lcp.py <cmd>

LCP_README.md    ← this file (human + agent readable)
```

---

## 與現有協議的比較 / Comparison with Existing Protocols

| | LCP v3 | MCP | ACP |
|---|---|---|---|
| 設計目標 / Target | Local 7B compressed comms | General tool calling | Agent collaboration |
| 訊息大小 / Msg size | 30–80 tokens | 150–500 tokens | 200–800 tokens |
| 指令複雜度 / Complexity | 6 fixed commands | Dynamic tool definitions | Dynamic action definitions |
| 7B 解析穩定度 / 7B stability | High (format-only) | Medium (semantic) | Low (complex structure) |
| 內建安全 / Built-in security | ✅ Anti-phishing | ❌ | Partial |
| 本地鎖定 / Local lock | ✅ | ❌ | ❌ |

> LCP 不是 MCP 的替代品，而是針對「本地 7B + 極端 token 限制」場景的專用協議。  
> LCP is not a replacement for MCP — it's a purpose-built protocol for the "local 7B + extreme token constraints" use case.

---

## 模型規模分析 / Model Size Analysis

**為什麼 3B 就夠用？/ Why 3B is sufficient:**

LCP 的設計刻意迴避了語意理解的需求。模型不需要「讀懂」任務，只需要「填格子」。

LCP is deliberately designed to avoid semantic understanding. The model doesn't need to "comprehend" the task — it just needs to "fill in the blanks."

```
傳統自然語言（需要語意）/ Traditional NL (needs semantics):
  "幫我查一下台北今天的天氣狀況，存起來，然後發文"
  → 模型要理解意圖、推斷步驟、組織輸出  ← 需要 7B+
  → Model must understand intent, infer steps, organize output

LCP 格式（純格式填空）/ LCP format (pattern fill):
  L|CA|openweather|taipei|E
  → 模型只需輸出固定格式  ← 3B 完全足夠
  → Model only outputs fixed pattern
```

**各規模模型實際極限 / Model Size Limits in Practice:**

```
7B   ✅ 完整穩定，推薦日常使用
        Full stability, recommended for daily use

3B   ✅ 穩定可用，建議搭配對照庫加速
        Stable, works best with translation store cache

1.5B ⚠️ 勉強可用，需要：
        Marginal, requires:
        - few-shot 範例在 system prompt
          few-shot examples in system prompt
        - 容錯 Parser（lcp.py 已內建）
          fault-tolerant parser (built into lcp.py)
        - 指令集縮減至 3–4 cmd
          reduce command set to 3–4 cmds

0.5B ❌ 格式遵循能力不足，不建議
        Insufficient format compliance, not recommended
```

**Token 消耗與模型大小無關 / Token count is model-size independent:**

```
token 數量由「訊息長度」決定，不是「模型大小」

Number of tokens is determined by message length, not model size.

L|CA|openweather|taipei|E   ← 3B 跑：~15 tokens
L|CA|openweather|taipei|E   ← 7B 跑：~15 tokens（一樣）

模型越大 → 推理越慢（不是 token 越多）
Larger model → slower inference (not more tokens)
```


---

## 授權 / License

MIT License — 自由使用與改作，請保留原始作者署名。  
Free to use and adapt. Attribution appreciated.

---

*This lobster was raised by hand. — Guoyu, 2026.03.14*
