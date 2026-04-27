# Entry Points 分析

## 主程式入口

### SKILL.md（Agent 宿主主入口）

**路徑**：`/SKILL.md:1-8`（frontmatter）

```yaml
name: dot-skill
description: "Unified meta-skill engine for distilling colleague, relationship, or celebrity characters..."
argument-hint: "[character] [name-or-slug]"
version: "1.0.0"
user-invocable: true
allowed-tools: Read, Write, Edit, Bash
```

這不是 Python CLI，而是一個 **Markdown 格式的 AI Agent 指令集**：
- 使用者在 Agent 宿主（Claude Code / Hermes / OpenClaw / Codex）輸入 `/dot-skill`
- Agent 宿主讀取 `SKILL.md`，把其中的 Markdown 指令作為系統提示注入 LLM
- LLM 按照 SKILL.md 的流程步驟，呼叫 Bash/Read/Write/Edit 工具完成操作

**觸發詞**（`SKILL.md:22-30`）：
- `/dot-skill`（標準）
- 「帮我创建一个 skill」
- 「我想蒸馏一个人」
- 「新建一个 skill」
- 「给我做一个 XX 的 skill」

**進化模式觸發**（`SKILL.md:39-47`）：
- 「我有新文件」/ 「追加」
- 「这不对」/ 「他不会这样」/ 「他应该是」
- `/update-skill {character} {slug}`
- `/update-colleague {slug}`（向後相容）

### Python CLI 入口

各個 Python 工具均有 `if __name__ == "__main__": main()` 入口：

| 工具 | 功能 | 主要參數 |
|------|------|---------|
| `tools/skill_writer.py:main()` | Skill 文件創建/更新/列出 | `--action [create\|update\|list]` |
| `tools/version_manager.py:main()` | 版本備份/回滾/清理 | `--action [list\|backup\|rollback\|cleanup]` |
| `tools/feishu_auto_collector.py` | 飛書自動採集 | `--name`, `--setup`, `--exchange-code` |
| `tools/dingtalk_auto_collector.py` | 釘釘自動採集 | `--name`, `--setup`, `--show-browser` |
| `tools/slack_auto_collector.py` | Slack 自動採集 | `--name`, `--setup` |
| `tools/feishu_parser.py` | 飛書 JSON 解析 | `--file`, `--target`, `--output` |
| `tools/email_parser.py` | 郵件解析 | `--file`, `--target`, `--output` |
| `tools/feishu_browser.py` | 飛書瀏覽器採集 | `--url`, `--target`, `--output` |
| `tools/feishu_mcp_client.py` | 飛書 MCP 採集 | `--url`, `--chat-id`, `--setup` |
| `tools/research/merge_research.py:main()` | 研究筆記合併 | `path`（skill 目錄） |
| `tools/research/quality_check.py` | Skill 品質驗證 | `skill_md_path`, `--profile` |
| `tools/research/srt_to_transcript.py` | 字幕轉文字 | `input`, `output` |
| `tools/research/transcribe_audio.py` | 音訊轉文字 | - |
| `tools/install_hermes_skill.py:main()` | Hermes 安裝（meta-skill） | `--source`, `--dest`, `--force`, `--dry-run` |
| `tools/install_openclaw_skill.py` | OpenClaw 安裝 | 同上 |
| `tools/install_codex_skill.py` | Codex 安裝 | 同上 |
| `tools/install_claude_generated_skill.py` | Claude 角色 Skill 安裝 | `--skill-dir`, `--force` |
| `tools/install_openclaw_generated_skill.py` | OpenClaw 角色 Skill 安裝 | 同上 |
| `tools/install_codex_generated_skill.py` | Codex 角色 Skill 安裝 | 同上 |

## 初始化序列（SKILL.md 執行流程）

```
使用者觸發 /dot-skill
    ↓
Agent 讀取 SKILL.md
    ↓
Step 0: 確認 character family（colleague / relationship / celebrity）
    ↓
Step 0.5: 若 celebrity，確認 research profile（budget-friendly / budget-unfriendly）
    ↓
Step 1: 讀取 prompts/{family}/intake.md → 引導基礎資訊錄入
    ↓
Step 2: 原材料導入（A/B/C/D/E 選項）
    → 飛書：python3 tools/feishu_auto_collector.py
    → 釘釘：python3 tools/dingtalk_auto_collector.py
    → Slack：python3 tools/slack_auto_collector.py
    → 文件：Read 工具直接讀取 PDF/圖片/MD
    → 郵件：python3 tools/email_parser.py
    ↓
Step 3: 分析原材料（LLM 讀取 analyzer prompts → 分析）
    → Work 線路：prompts/work_analyzer.md
    → Persona 線路：prompts/{family}/persona_analyzer.md
    ↓
Step 4: 生成預覽（LLM 讀取 builder prompts → 生成草稿）
    → prompts/work_builder.md
    → prompts/{family}/persona_builder.md
    ↓
Step 5: 寫入文件（通過 skill_writer.py）
    → python3 tools/skill_writer.py --action create ...
    → 輸出：skills/{character}/{slug}/ 下的完整 artifact 集
```

## skill_writer.py 初始化序列

`tools/skill_writer.py:create_skill()` 函式（第 226-249 行）：

1. 呼叫 `enrich_skill_meta()` 正規化 meta
2. 取得 character preset（`get_character_preset()`）
3. 建立目錄：`skills/{character}/{slug}/`、`versions/`、`knowledge/docs|messages|emails/`
4. 設定 lifecycle 欄位（created_at, updated_at, version=v1）
5. 呼叫 `write_artifacts()` 寫出所有 artifact 文件
