# 外部整合分析

## 1. 飛書（Feishu）API 整合

**工具：** `tools/feishu_auto_collector.py`, `tools/feishu_parser.py`, `tools/feishu_browser.py`, `tools/feishu_mcp_client.py`

### API 端點

| 端點 | 用途 | 認證方式 |
|------|------|---------|
| `POST /auth/v3/app_access_token/internal` | 獲取 tenant_access_token | app_id + app_secret |
| `POST /authen/v1/oidc/access_token` | 用 OAuth code 換 user_access_token | app_access_token + code |
| `GET /im/v1/messages?container_id_type=chat&container_id={chat_id}` | 拉取群聊訊息 | tenant_access_token |
| `POST /im/v1/messages?receive_id_type=open_id` | 發訊息（用於取得 p2p chat_id） | user_access_token |
| `GET /contact/v3/scopes` | 取得通訊錄可見範圍 | tenant_access_token |
| `GET /contact/v3/users/{user_id}` | 查詢用戶資訊 | tenant_access_token |

### 認證流程

**群聊採集（tenant_access_token）：**
1. App ID + App Secret → POST `/auth/v3/app_access_token/internal` → tenant_access_token
2. 使用 tenant_access_token 直接呼叫 IM API

**私聊採集（user_access_token）：**
1. 生成 OAuth URL（`/authen/v1/authorize?app_id=...&redirect_uri=...`）
2. 用戶在瀏覽器授權，從 redirect URL 取得 code
3. code + app_access_token → POST `/authen/v1/oidc/access_token` → user_access_token（2小時有效）
4. 使用 user_access_token 呼叫私聊 API

### 瀏覽器方案
使用 playwright 複用 Chrome 本機登入態，繞過 API token 限制，適用於無 App 權限的內部文件。

### 失敗處理
- `SKILL.md:252-258`：根據報錯診斷原因，常見：bot 未入群、token 過期（user token 2小時有效，需 refresh_token 刷新）、權限不足
- 失敗時建議換用方式 B/C

---

## 2. 釘釘（DingTalk）API 整合

**工具：** `tools/dingtalk_auto_collector.py`

### 特殊限制
釘釘 API 不支援歷史訊息拉取（`SKILL.md:289`），訊息採集自動切換為瀏覽器採集。

### 採集內容
- 文件和知識庫（API 支援）
- 多維表格（API 支援）
- 訊息記錄（瀏覽器採集，使用 playwright）

### 認證
使用釘釘開放平台的 AppKey + AppSecret，透過 `--setup` 初次設定。首次使用需 `--show-browser` 完成登入。

---

## 3. Slack API 整合

**工具：** `tools/slack_auto_collector.py`
**依賴：** `slack-sdk >= 3.27.0`

### 所需 Bot Token Scopes

| Scope | 必要性 |
|-------|--------|
| `users:read` | 必需（搜尋用戶） |
| `channels:read` + `channels:history` | 必需（public channel） |
| `groups:read` + `groups:history` | 必需（private channel） |
| `mpim:read` + `mpim:history` | 可選（群 DM） |
| `im:read` + `im:history` | 可選（1:1 DM） |

### 速率限制
slack-sdk 自動處理 429 重試，無需手動干預（`INSTALL.md:293`）。

### 已知限制
- 免費 Workspace：僅能存取最近 90 天訊息
- Bot 必須已加入目標頻道（`/invite @bot`）

---

## 4. 研究工具鏈（Celebrity 子系統）

**工具：** `tools/research/download_subtitles.sh`, `tools/research/srt_to_transcript.py`, `tools/research/transcribe_audio.py`

### download_subtitles.sh
- 使用外部影片字幕下載工具（⚠️ 具體 CLI 工具未在程式碼中說明，推測為 yt-dlp 或 youtube-dl）
- 輸出：SRT 字幕文件

### srt_to_transcript.py
- 輸入：SRT 字幕文件
- 輸出：清理後的 transcript 文字（去除時間戳記、格式化）

### transcribe_audio.py
- 音訊轉文字（⚠️ 具體使用的轉寫 API 未在採集中確認，可能是 whisper 或其他）

---

## 5. Agent 宿主整合

### Claude Code

**安裝目標：**
- Meta-skill：`~/.claude/skills/dot-skill/`
- 生成的角色 Skill：`~/.claude/skills/{character}-{slug}/`
- Windows 相容 shim：`~/.claude/commands/{character}-{slug}.md`

**工具：** `tools/install_claude_generated_skill.py`

Windows 相容性：由於 Claude Code 在 Windows 上的 skill 發現問題，安裝器會額外寫入 commands 目錄下的 `.md` shim 文件（`INSTALL.md:54`）。

### Hermes

**安裝目標：** `~/.hermes/skills/openclaw-imports/dot-skill/`

**工具：** `tools/install_hermes_skill.py`
- 使用 `shutil.copytree` 複製整個 repo（排除 `.git`, `__pycache__`, `.DS_Store`, `*.pyc`）
- 支援 `--dry-run` 預覽

### OpenClaw

**安裝目標：** `~/.openclaw/workspace/skills/dot-skill/`

**工具：** `tools/install_openclaw_skill.py`

### Codex

**安裝目標：** `~/.codex/skills/dot-skill/`

**工具：** `tools/install_codex_skill.py`

Codex 無 slash command 機制，生成的角色 Skill 以 `{character}-{slug}` 的 skill 名稱安裝（`INSTALL.md:116`）。

---

## 6. Email 整合

**工具：** `tools/email_parser.py`

支援格式：`.eml` / `.mbox`

解析後提取目標人物（`--target`）的發件內容，輸出為純文字供 LLM 分析。

---

## 7. Feishu MCP 整合

**工具：** `tools/feishu_mcp_client.py`
**依賴：** `feishu-mcp`（npm 套件，需 Node.js 16+）

通過飛書官方 MCP（Model Context Protocol）伺服器讀取文件和訊息，適用於有公司授權的文件場景。

---

## 失敗處理總覽

| 整合 | 失敗場景 | 處理策略 |
|------|---------|---------|
| 飛書 API | Bot 未入群 | 提示加入 Bot 後重試 |
| 飛書 API | user_token 過期 | 提示用 refresh_token 刷新 |
| 飛書 API | 權限不足 | 引導用戶在開放平台授權 |
| 釘釘 | 訊息 API 不支援 | 自動切換瀏覽器採集 |
| 釘釘 | 訊息採集失敗 | 提示截圖上傳 |
| Slack | 速率限制 | SDK 自動等待重試 |
| Slack | 未入頻道 | 提示 `/invite @bot` |
| Celebrity 研究 | 平台驗證拦截 | 保留已有筆記，記錄限制原因，繼續生成但標記 source_grounding 未完成；嚴禁編造 URL |
| 任意採集器 | 通用失敗 | 可改用其他採集方式（A/B/C/D/E 均可混用） |
