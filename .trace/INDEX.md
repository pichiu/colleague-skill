# dot-skill 技術文件索引

## 一段話總結

dot-skill（前身 colleague-skill）是一個運行在 AI Agent 宿主（Claude Code / Hermes / OpenClaw / Codex）上的 **meta-skill 引擎**：使用者提供某人的原材料（聊天記錄、文件、郵件）加上描述，系統自動生成一個可複用的 AI「人物 Skill」——讓 AI 以目標人物的思維框架、語言風格、工作方法來工作。目標受眾是想留存同事知識、維繫情感連結、或深度研究公眾人物的個人用戶。

---

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | 3.9+ | 所有工具腳本 |
| 文件格式 | Markdown | - | Prompt 模板、SKILL.md 入口、所有文件 |
| 資料格式 | JSON | - | meta.json、manifest.json |
| Agent 標準 | AgentSkills | - | 整個 repo 是一個標準 skill 目錄 |
| Agent Host | Claude Code | - | 原生 slash command 支援 |
| Agent Host | Hermes | - | 一鍵安裝，/dot-skill 入口 |
| Agent Host | OpenClaw | - | 完全相容 |
| Agent Host | Codex | - | 以 skill 名稱調用 |
| HTTP 客戶端 | requests | >=2.28.0 | Feishu/Slack API 呼叫 |
| 瀏覽器自動化 | playwright | >=1.40.0 | Feishu 瀏覽器採集、釘釘訊息採集（可選） |
| Slack 整合 | slack-sdk | >=3.27.0 | Slack 自動採集（可選） |
| 中文轉 slug | pypinyin | >=0.48.0 | 中文姓名轉拼音（可選） |
| Office 解析 | python-docx + openpyxl | >=1.1.0 + >=3.1.0 | Word/Excel 解析（可選） |
| CI | GitHub Actions | - | Python 3.9/3.11 unittest + ruff lint |

---

## 關鍵指令速查

```bash
# 安裝到 Claude Code（全局）
git clone https://github.com/titanwings/colleague-skill ~/.claude/skills/dot-skill

# 安裝到 Hermes
python3 tools/install_hermes_skill.py --force

# 安裝到 OpenClaw
python3 tools/install_openclaw_skill.py --force

# 安裝到 Codex
python3 tools/install_codex_skill.py --force

# 啟動 dot-skill
/dot-skill    # 在任意支援的 Agent 宿主中輸入

# 列出已生成的 Skills
python3 tools/skill_writer.py --action list --character colleague --base-dir ./skills/colleague
python3 tools/skill_writer.py --action list --character relationship --base-dir ./skills/relationship
python3 tools/skill_writer.py --action list --character celebrity --base-dir ./skills/celebrity

# 回滾 Skill 版本
python3 tools/version_manager.py --action rollback --character colleague --slug {slug} --version v1 --base-dir ./skills/colleague

# 飛書採集初始化
python3 tools/feishu_auto_collector.py --setup

# Slack 採集初始化
python3 tools/slack_auto_collector.py --setup

# Celebrity 研究工具鏈
bash tools/research/download_subtitles.sh "<url>" "./tmp/subtitles"
python3 tools/research/srt_to_transcript.py "./tmp/subtitles/example.srt"
python3 tools/research/merge_research.py "./skills/celebrity/<slug>"
python3 tools/research/quality_check.py "./skills/celebrity/<slug>/SKILL.md"

# 執行測試
python -m unittest discover -s tests -p 'test_*.py' -v

# 靜態分析
ruff check tools/
```

---

## 文件地圖

| 文件 | 位置 | 說明 |
|------|------|------|
| 專案總覽（本文件） | `.trace/INDEX.md` | 快速上手的起點 |
| 程式碼地圖 | `.trace/CODEBASE_MAP.md` | 目錄結構 + 「我想改 X 要看哪裡」速查表 |
| 系統架構 | `.trace/ARCHITECTURE.md` | 架構圖、元件說明、設計決策 |
| 資料模型 | `.trace/DATA_MODEL.md` | Skill artifact 結構、meta.json schema、狀態機 |
| API / CLI 參考 | `.trace/API_SURFACE.md` | 所有 CLI 指令、prompt 介面、生成 Skill 調用方式 |
| 開發者上手指南 | `.trace/DEV_GUIDE.md` | 本地設定、測試、貢獻流程 |
| 探索紀錄 | `.trace/DISCOVERY_LOG.md` | 發現的問題、技術債、待解答問題 |
| 官方 README | `README.md` | 功能介紹、Demo、使用說明（英文） |
| 官方安裝指南 | `INSTALL.md` | 各 host 安裝步驟、Slack 設定 |
| PRD | `docs/PRD.md` | 原始產品需求文件（部分已過期，見 DISCOVERY_LOG） |
| Roadmap | `ROADMAP.md` | 產品路線圖 |

---

## 專案專屬術語表

| 術語 | 定義 |
|------|------|
| **dot-skill** | 本專案的正式名稱（v1.0.0），前身為 colleague-skill |
| **meta-skill** | 用於生成其他 Skill 的 Skill（dot-skill 本身即是 meta-skill） |
| **character family** | 蒸餾目標的類別：`colleague`（同事）、`relationship`（親密關係）、`celebrity`（公眾人物） |
| **Skill** | 符合 AgentSkills 標準的 AI 人物檔案，由 SKILL.md + work.md + persona.md 等組成 |
| **Work Skill** | Skill 的工作能力部分，讓 AI 用目標人物的技術方法工作 |
| **Persona** | Skill 的性格部分，讓 AI 用目標人物的溝通風格回應 |
| **蒸餾（Distill）** | 將人的知識、性格、語言風格提煉成 AI Skill 的過程 |
| **slug** | Skill 的唯一識別符（URL 友好格式，如 `zhangsan`、`andrej-karpathy`） |
| **Layer 0** | Persona 結構中優先級最高的硬覆蓋層，對應使用者手動輸入的標籤 |
| **Correction** | 對話中用戶糾正 Skill 行為的機制，寫入 Layer 5 Correction 記錄 |
| **research profile** | Celebrity family 的研究深度配置：`budget-friendly`（輕量）或 `budget-unfriendly`（深度） |
| **manifest.json** | Skill 的機器可讀元數據，供安裝器和 Gallery 使用 |
| **meta.json** | Skill 的完整元數據，包含版本歷史、生成設定、來源記錄 |
| **artifact** | Skill 目錄下的一個輸出文件（SKILL.md、work.md、persona.md 等） |
| **budget-unfriendly** | Celebrity 深度研究模式，含六軌獨立研究筆記、審計、合成、驗證流程 |
| **六維度研究** | Celebrity 的研究框架：著作、對話、表達 DNA、決策、他者視角、時間線 |
| **AgentSkills** | dot-skill 遵循的 Agent Skill 開放標準（agentskills.io） |
| **tensor_access_token / user_access_token** | 飛書 API 的兩種認證 token（應用身份 vs 用戶身份） |
