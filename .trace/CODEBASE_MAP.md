# dot-skill 程式碼地圖

## Annotated Directory Tree

```
colleague-skill/
│
├── SKILL.md                          # ⭐ 主入口：AgentSkills 格式，1464 行雙語指令集
│                                     #    frontmatter: name=dot-skill, version=1.0.0
│                                     #    觸發：/dot-skill（任意 Agent 宿主）
│
├── requirements.txt                  # Python 依賴（requests 必選，其餘可選）
│
├── prompts/                          # ⭐ Prompt 模板庫（純 Markdown，不執行）
│   ├── intake.md                     #    [colleague] 引導式資訊錄入腳本
│   ├── work_analyzer.md              #    [shared] Work 分析：5 維度提取框架
│   ├── persona_analyzer.md           #    [colleague] Persona 分析：4 維度 + 標籤翻譯表
│   ├── work_builder.md               #    [shared] Work 生成模板
│   ├── persona_builder.md            #    [colleague] Persona 生成：6 層結構模板
│   ├── merger.md                     #    [shared] 增量合併邏輯
│   ├── correction_handler.md         #    [shared] 對話糾正識別與應用
│   ├── relationship/                 #    [relationship family] 情感類 prompts
│   │   ├── intake.md
│   │   ├── persona_analyzer.md       #    情感/情緒/衝突/修復模式分析
│   │   ├── persona_builder.md
│   │   └── merger.md
│   └── celebrity/                    #    [celebrity family] 公眾人物 prompts
│       ├── intake.md
│       ├── research.md               #    六維度研究策略（budget-friendly）
│       ├── persona_analyzer.md       #    含 mental models / expression DNA 分析
│       ├── persona_builder.md
│       ├── merger.md
│       └── budget_unfriendly/        #    [budget-unfriendly] 深度研究子系統
│           ├── research.md           #       六軌獨立研究策略
│           ├── audit.md              #       研究審計（PASS/FAIL）
│           ├── synthesis.md          #       mental models triple-gate 合成
│           ├── validation.md         #       已知答案驗證
│           ├── persona_analyzer.md
│           └── persona_builder.md
│
├── tools/                            # ⭐ Python 工具層（CLI + library）
│   │
│   ├── skill_writer.py               # ⭐ 核心 I 型：Skill 生命週期管理
│   │                                 #    create_skill(), update_skill(), list_skills()
│   │                                 #    merge_markdown_patch(), apply_correction()
│   │                                 #    main() --action [create|update|list]
│   │
│   ├── skill_schema.py               # ⭐ 核心 II 型：元數據 Schema
│   │                                 #    enrich_skill_meta()：向後相容元數據正規化
│   │                                 #    build_manifest()：機器可讀 manifest 生成
│   │                                 #    build_artifact_names()：命名策略
│   │
│   ├── skill_presets.py              # ⭐ 核心 III 型：Character Preset 倉庫
│   │                                 #    CHARACTER_PRESETS：3 個 family 的完整設定
│   │                                 #    CHARACTER_ALIASES：別名映射
│   │                                 #    resolve_storage_root()：路徑解析
│   │
│   ├── version_manager.py            #    版本管理：backup/rollback/cleanup
│   │                                 #    MAX_VERSIONS = 10
│   │
│   ├── feishu_auto_collector.py      #    飛書採集（群聊+私聊 API）
│   ├── feishu_browser.py             #    飛書採集（playwright 瀏覽器方案）
│   ├── feishu_mcp_client.py          #    飛書採集（MCP App Token 方案）
│   ├── feishu_parser.py              #    飛書 JSON 導出解析
│   ├── dingtalk_auto_collector.py    #    釘釘採集（API + playwright 瀏覽器）
│   ├── slack_auto_collector.py       #    Slack 採集（slack-sdk）
│   ├── email_parser.py               #    郵件 .eml/.mbox 解析
│   │
│   ├── install_hermes_skill.py       #    Hermes meta-skill 安裝器
│   ├── install_openclaw_skill.py     #    OpenClaw meta-skill 安裝器
│   ├── install_codex_skill.py        #    Codex meta-skill 安裝器
│   ├── install_claude_generated_skill.py    # Claude Code 角色 Skill 安裝器
│   ├── install_openclaw_generated_skill.py  # OpenClaw 角色 Skill 安裝器
│   ├── install_codex_generated_skill.py     # Codex 角色 Skill 安裝器
│   ├── install_generated_skill_common.py    # 安裝器共用邏輯
│   │
│   └── research/                     #    [celebrity] 研究工具鏈
│       ├── download_subtitles.sh     #       影片字幕下載
│       ├── transcribe_audio.py       #       音訊轉文字
│       ├── srt_to_transcript.py      #       SRT 字幕→ transcript
│       ├── merge_research.py         #       研究筆記合併 + 品質指標統計
│       ├── quality_check.py          #       Skill 品質驗證（PASS/FAIL）
│       └── __init__.py
│
├── skills/                           # 生成的 Skill 產物（.gitignore 排除，不進版控）
│   ├── colleague/
│   │   └── {slug}/                   # 每個同事 Skill
│   │       ├── SKILL.md              #   合併版（Work + Persona + 運行規則）
│   │       ├── work.md               #   Work 原始內容
│   │       ├── persona.md            #   Persona 原始內容
│   │       ├── work_skill.md         #   Work-only Skill（帶 frontmatter）
│   │       ├── persona_skill.md      #   Persona-only Skill（帶 frontmatter）
│   │       ├── manifest.json         #   機器可讀元數據
│   │       ├── meta.json             #   完整元數據（含版本、標籤）
│   │       ├── versions/             #   歷史版本（最多 10 個）
│   │       └── knowledge/            #   原始採集材料
│   │           ├── docs/
│   │           ├── messages/
│   │           └── emails/
│   ├── relationship/                 # 同結構
│   └── celebrity/
│       └── {slug}/
│           └── knowledge/
│               ├── docs/
│               ├── messages/
│               ├── emails/
│               ├── research/raw/     # 六維度研究筆記
│               ├── research/merged/  # 合併後的 summary.md
│               ├── research/reviews/ # audit/synthesis/validation（budget-unfriendly）
│               ├── transcripts/      # 字幕轉 transcript
│               └── subtitles/        # 原始字幕文件
│
├── references/                       # Celebrity budget-unfriendly 框架文件
├── docs/                             # 文件（PRD、設計文件、多語言 README）
├── tests/                            # 單元測試（7 個測試文件）
└── .github/workflows/ci.yml          # CI：Python 3.9/3.11 + ruff
```

---

## 「我想改 X 要看哪裡？」速查表

| 我想要... | 看這裡 | 關鍵檔案 |
|----------|--------|---------|
| 新增一個 character family | `tools/skill_presets.py` + `prompts/` | `skill_presets.py:CHARACTER_PRESETS` |
| 修改 colleague 的 Persona 分析框架 | `prompts/` | `prompts/persona_analyzer.md` |
| 修改 Work Skill 的提取維度 | `prompts/` | `prompts/work_analyzer.md` |
| 修改生成的 SKILL.md 結構/語言 | `tools/skill_writer.py` | `SKILL_MD_TEMPLATE_ZH` / `SKILL_MD_TEMPLATE_EN`（第 37-108 行） |
| 新增一個 Agent 宿主支援 | `tools/` + `tools/skill_writer.py` | `install_{host}_skill.py` + `install_generated_hosts()` |
| 新增一個資料採集器 | `tools/` | `{source}_auto_collector.py` + 更新 `SKILL.md` Step 2 |
| 修改版本保留數量 | `tools/version_manager.py` | `MAX_VERSIONS = 10`（第 21 行） |
| 修改 Skill 的命名策略 | `tools/skill_schema.py` | `build_artifact_names()`（第 103 行） |
| 修改 slug 生成邏輯 | `tools/skill_writer.py` | `slugify()`（第 111 行） |
| 修改 celebrity 品質門檻 | `tools/skill_presets.py` | `research_profiles.budget-friendly.min_*`（第 117 行） |
| 修改 Markdown patch 合併邏輯 | `tools/skill_writer.py` | `merge_markdown_patch()`（第 264 行） |
| 修改糾正記錄格式 | `tools/skill_writer.py` | `apply_correction()`（第 302 行） |
| 修改元數據 schema | `tools/skill_schema.py` | `enrich_skill_meta()`（第 153 行） |
| 新增 celebrity research profile | `tools/skill_presets.py` | `celebrity.research_profiles`（第 109 行） |
| 修改預設 storage root | `tools/skill_presets.py` | `storage_root` 欄位（第 35, 65, 93 行） |
| 控制 Claude 自動安裝行為 | 環境變數 | `DOT_SKILL_AUTO_INSTALL_CLAUDE=0` |
| 新增語言模板（現有：中/英） | `tools/skill_writer.py` | `prefers_chinese()` + 新增 template 變數 |

---

## 模組依賴關係圖

```mermaid
graph TD
    A[SKILL.md<br/>主入口] -->|引用| B[prompts/\n各 family 模板]
    A -->|呼叫 Bash| C[tools/\n採集器]
    A -->|呼叫 Bash| D[tools/skill_writer.py\n核心 I]
    A -->|呼叫 Bash| E[tools/version_manager.py\n版本管理]
    A -->|呼叫 Bash| F[tools/research/\nCelebrity 工具鏈]

    D -->|import| G[tools/skill_schema.py\n核心 II]
    D -->|import| H[tools/skill_presets.py\n核心 III]
    D -->|import| I[tools/install_*_generated_skill.py\n安裝器]

    E -->|import| H
    E -->|import| G

    G -->|import| H

    C -->|輸出 txt/json| J[knowledge/\n原始材料]
    D -->|寫出| K[skills/{character}/{slug}/\n完整 artifact 集]

    I -->|複製到| L[~/.claude/skills/<br/>~/.openclaw/workspace/skills/<br/>~/.codex/skills/]

    subgraph "Core Triad"
        D
        G
        H
    end
```
