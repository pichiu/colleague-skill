# Stage 1 偵察報告

## 1.1 技術棧

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | 3.9+ | 工具腳本執行環境 |
| 語言 | Python | 3.9+ | 所有 `tools/*.py` 主力語言 |
| 輔助語言 | Bash | - | 字幕下載工具 `download_subtitles.sh` |
| 文件格式 | Markdown | - | Prompt 模板、SKILL.md 入口、所有文件 |
| 資料格式 | JSON | - | meta.json、manifest.json、correction 輸入 |
| 套件 | requests | >=2.28.0 | HTTP API 呼叫（Feishu、Slack 等） |
| 套件 | pypinyin | >=0.48.0 | 中文姓名轉拼音 slug（可選） |
| 套件 | playwright | >=1.40.0 | Feishu 瀏覽器登入態採集（可選） |
| 套件 | slack-sdk | >=3.27.0 | Slack 自動採集（可選） |
| 套件 | python-docx | >=1.1.0 | Word 文件解析（可選） |
| 套件 | openpyxl | >=3.1.0 | Excel 解析（可選） |
| CI/CD | GitHub Actions | - | 測試（unittest）+ 靜態分析（ruff，非阻塞） |
| Agent Hosts | Claude Code / Hermes / OpenClaw / Codex | - | 運行 SKILL.md 的 AI Agent 宿主 |
| 標準 | AgentSkills | - | 整個 repo 是一個 AgentSkills 標準 skill 目錄 |

## 1.2 目錄結構（3 層深度）

```
colleague-skill/
├── SKILL.md                          # 主入口：dot-skill meta-skill 完整指令集（雙語，1464行）
├── README.md                         # 項目說明（英文）+ 技術棧、用法、demo
├── INSTALL.md                        # 安裝指南（4 個 host + 依賴安裝）
├── ROADMAP.md                        # 路線圖（英文）
├── SKILL.md                          # AgentSkills 標準格式入口
├── CONTRIBUTING.md                   # 貢獻指南
├── LICENSE                           # MIT 授權
├── requirements.txt                  # Python 依賴（必選 requests，其餘可選）
├── colleague_skill.pdf               # 技術報告 PDF（前身 colleague.skill 論文）
├── openarena-claim.txt               # ⚠️ 未知用途的文字文件
│
├── prompts/                          # Prompt 模板庫（不直接執行，供 SKILL.md 引用）
│   ├── intake.md                     # [colleague] 基礎資訊錄入腳本
│   ├── work_analyzer.md              # [colleague/shared] Work 分析 prompt
│   ├── persona_analyzer.md           # [colleague] Persona 分析 prompt
│   ├── work_builder.md               # [colleague/shared] Work 生成模板
│   ├── persona_builder.md            # [colleague] Persona 生成模板
│   ├── merger.md                     # [shared] 增量合併 prompt
│   ├── correction_handler.md         # [shared] 對話糾正處理
│   ├── relationship/                 # [relationship] 情感關係 prompts
│   │   ├── intake.md
│   │   ├── persona_analyzer.md
│   │   ├── persona_builder.md
│   │   └── merger.md
│   └── celebrity/                    # [celebrity] 公眾人物 prompts
│       ├── intake.md
│       ├── research.md               # 六維度研究策略（budget-friendly）
│       ├── persona_analyzer.md
│       ├── persona_builder.md
│       ├── merger.md
│       └── budget_unfriendly/        # [celebrity/budget-unfriendly] 深度研究
│           ├── research.md
│           ├── audit.md
│           ├── synthesis.md
│           ├── validation.md
│           ├── persona_analyzer.md
│           └── persona_builder.md
│
├── tools/                            # Python 工具腳本
│   ├── skill_writer.py               # 核心：Skill 文件創建/更新/列出
│   ├── skill_schema.py               # Schema 輔助：元數據正規化、artifact 命名
│   ├── skill_presets.py              # Character 預設倉庫（colleague/relationship/celebrity）
│   ├── version_manager.py            # 版本備份與回滾
│   ├── feishu_auto_collector.py      # 飛書自動採集（API）
│   ├── feishu_browser.py             # 飛書瀏覽器採集（playwright）
│   ├── feishu_mcp_client.py          # 飛書 MCP 客戶端（App Token）
│   ├── feishu_parser.py              # 飛書 JSON 導出解析
│   ├── dingtalk_auto_collector.py    # 釘釘自動採集（API + 瀏覽器）
│   ├── slack_auto_collector.py       # Slack 自動採集（slack-sdk）
│   ├── email_parser.py               # 郵件 .eml/.mbox 解析
│   ├── install_hermes_skill.py       # Hermes 宿主安裝器（meta-skill）
│   ├── install_openclaw_skill.py     # OpenClaw 宿主安裝器（meta-skill）
│   ├── install_codex_skill.py        # Codex 宿主安裝器（meta-skill）
│   ├── install_claude_generated_skill.py    # Claude Code 已生成角色 Skill 安裝
│   ├── install_openclaw_generated_skill.py  # OpenClaw 已生成角色 Skill 安裝
│   ├── install_codex_generated_skill.py     # Codex 已生成角色 Skill 安裝
│   ├── install_generated_skill_common.py    # 安裝器共用邏輯
│   └── research/                     # [celebrity] 研究工具鏈
│       ├── download_subtitles.sh     # 影片字幕下載
│       ├── transcribe_audio.py       # 音訊轉文字
│       ├── srt_to_transcript.py      # 字幕轉 transcript
│       ├── merge_research.py         # 六維度研究筆記合併
│       ├── quality_check.py          # Skill 品質驗證
│       └── __init__.py
│
├── skills/                           # 生成的 Skill 產物（.gitignore 排除）
│   ├── colleague/                    # 同事類 Skills
│   │   ├── example_zhangsan/         # 範例：張三
│   │   ├── example_tianyi/           # 範例：天一
│   │   └── example_jiaxiu/           # 範例：嘉秀
│   ├── relationship/                 # 關係類 Skills
│   └── celebrity/                    # 公眾人物類 Skills
│
├── references/                       # 參考資料
│   ├── celebrity_budget_unfriendly_framework.md   # budget-unfriendly 框架說明
│   └── celebrity_budget_unfriendly_template.md    # budget-unfriendly 模板
│
├── docs/                             # 文件
│   ├── PRD.md                        # 產品需求文件 v2.0（原 colleague.skill 規格）
│   ├── SKILL_TYPE_ABSTRACTION_DESIGN.md   # Skill 類型抽象設計（英文）
│   ├── SKILL_TYPE_ABSTRACTION_DESIGN_ZH.md  # Skill 類型抽象設計（中文）
│   ├── assets/                       # WeChat 社群 QR 碼圖片
│   └── lang/                         # 多語言版 README 和 ROADMAP（7 種語言）
│
├── tests/                            # 單元測試
│   ├── test_cli_lifecycle.py
│   ├── test_install_claude_generated_skill.py
│   ├── test_install_hermes_skill.py
│   ├── test_install_openclaw_and_codex.py
│   ├── test_research_tools.py
│   ├── test_skill_entrypoint_docs.py
│   └── test_skill_writer.py
│
└── .github/
    ├── workflows/ci.yml              # CI: test (py3.9+py3.11) + ruff lint
    └── ISSUE_TEMPLATE/               # Bug/Feature/Question 模板
```

## 1.3 架構模式識別

**Plugin-based meta-skill 架構**

- 整個 repo 是一個符合 AgentSkills 標準的 skill 目錄，可被 clone 到各種 AI Agent host
- SKILL.md 是「控制平面」，提供 AI Agent 完整的執行流程指令
- Character 系統（colleague/relationship/celebrity）採用 preset 模式，可擴展新類型而不改核心
- Prompt 模板庫是獨立的「知識庫」，SKILL.md 動態引用
- Python tools 是「執行層」，負責具體的 I/O 操作

## 1.4 既有文件摘要

### docs/PRD.md
- 原始產品需求文件 v2.0，描述 colleague.skill 的 MVP 規格
- 文件說 `/create-colleague` → 實際程式碼已升級為 `/dot-skill`（落差）
- 文件說輸出目錄是 `~/.openclaw/workspace/skills/colleagues/` → 實際是 `./skills/colleague/`（落差）
- 仍有參考價值：定義了 5 層 Persona 結構、Work Skill 5 個提取維度

### docs/SKILL_TYPE_ABSTRACTION_DESIGN*.md
- 描述從 colleague-only 到三類 character family 的抽象化設計
- 定義了 character preset 系統的設計意圖

## 1.5 文件與程式碼落差分析

| 落差項目 | 文件說 | 程式碼實際 | 位置 |
|----------|--------|-----------|------|
| 主入口命令 | `/create-colleague` | `/dot-skill` | `PRD.md` vs `SKILL.md:line 37` |
| 輸出目錄 | `~/.openclaw/workspace/skills/colleagues/` | `./skills/colleague/` | `PRD.md:七` vs `skill_presets.py:line 35` |
| Layer 設計 | 5 層（Layer 0-4） | 5 層 + Correction 記錄（實際是 6 段） | `PRD.md:5.2` vs `persona_builder.md` |
| 版本限制 | 最多 10 個版本 | `MAX_VERSIONS = 10`（已實作） | `PRD.md:6.3` vs `version_manager.py:line 21` |
| P0 實作狀態 | 文件標記 `[ ]` 未完成 | 實際已完整實作 | `PRD.md:九` |

## 1.6 統計數據

- 總檔案數：125 個（不含 .git）
- Python 原始碼：約 18 個 .py 檔
- Markdown 文件/Prompt：約 30 個 .md 檔
- 測試檔案：7 個
- 規模評估：小型專案，無需縮限 trace 範圍
