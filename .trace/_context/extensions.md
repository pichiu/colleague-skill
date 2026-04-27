# Extension Points 分析

## 1. Character Family Preset 擴展點

**位置：** `tools/skill_presets.py:CHARACTER_PRESETS`（第 23 行）

這是最主要的擴展點。新增 character family 的步驟：

1. 在 `CHARACTER_PRESETS` 字典中新增一個 key-value
2. 在 `prompts/` 下建立對應目錄，放入 prompt 文件
3. 在 `skills/` 下，對應目錄會由 `skill_writer.py` 自動建立

```python
# 示例：新增 "self" family（蒸餾自己）
CHARACTER_PRESETS["self"] = {
    "character": "self",
    "prompt_bundle": {
        "preset": "dot.self.v1",
        "intake": "prompts/self/intake.md",
        "persona_analyzer": "prompts/self/persona_analyzer.md",
        "persona_builder": "prompts/self/persona_builder.md",
        ...
    },
    ...
}
```

已有 alias 機制支援別名（`CHARACTER_ALIASES` 第 170-178 行）：
```python
CHARACTER_ALIASES = {
    "ex": "relationship",
    "self": "relationship",    # 目前指向 relationship
    "icon": "celebrity",
    "fictional-character": "celebrity",
}
```

## 2. Research Profile 擴展點（Celebrity 子系統）

**位置：** `skill_presets.py:celebrity.research_profiles`（第 109-165 行）

celebrity family 的研究深度可以用 research profile 擴展：

- `budget-friendly`：輕量公開來源蒸餾（3+ raw notes, 2+ URLs）
- `budget-unfriendly`：深度六軌研究（6+ raw notes, 8+ URLs, audit+synthesis+validation）

新增 research profile 的步驟：
1. 在 `celebrity.research_profiles` 中新增 key
2. 定義 `prompt_bundle`（指向對應的 prompts）
3. 定義品質門檻（`min_raw_notes`, `min_grounded_urls` 等）
4. 可選：在 `prompts/celebrity/` 下建立對應子目錄

## 3. Prompt 模板替換點

**位置：** `prompts/` 目錄下所有 `.md` 檔案

Prompt 模板是純 Markdown 文件，完全解耦於 Python 工具層。可替換或擴展以下模板：

| Prompt | 角色 | 替換影響 |
|--------|------|---------|
| `prompts/intake.md` | colleague 資訊錄入 | 改變使用者互動問題 |
| `prompts/work_analyzer.md` | Work 分析邏輯 | 改變提取的工作維度 |
| `prompts/persona_analyzer.md` | Persona 分析邏輯 | 改變性格分析框架 |
| `prompts/work_builder.md` | Work 生成模板 | 改變輸出的 work.md 結構 |
| `prompts/persona_builder.md` | Persona 生成模板 | 改變 Persona 六層結構 |
| `prompts/merger.md` | 增量合併邏輯 | 改變 Skill 演化策略 |
| `prompts/correction_handler.md` | 對話糾正識別 | 改變糾正解析邏輯 |

## 4. Host 安裝器擴展點

**位置：** `tools/install_*_skill.py`

目前支援 4 個 Agent 宿主（Claude Code / Hermes / OpenClaw / Codex），每個宿主有：
- 一個 meta-skill 安裝器（安裝 dot-skill 本身）
- 一個 generated skill 安裝器（安裝已生成的角色 Skill）

新增宿主的步驟：
1. 建立 `tools/install_{newhost}_skill.py`（meta-skill 安裝器）
2. 建立 `tools/install_{newhost}_generated_skill.py`（角色 Skill 安裝器）
3. 在 `skill_writer.py:install_generated_hosts()` 加入對應 if block（第 434-493 行）
4. 在 `skill_writer.py:main()` 加入對應 argparse 參數
5. 在 `skill_schema.py:build_manifest()` 的 `installers` 加入映射（第 289 行）

## 5. 資料採集器擴展點

**位置：** `tools/` 目錄下的採集工具

目前支援的採集源：Feishu / DingTalk / Slack / Email / PDF / 圖片 / WeChat（SQLite）

新增採集器的步驟：
1. 建立 `tools/{source}_auto_collector.py`（支援 `--setup`, `--name`, `--output-dir`）
2. 在 `SKILL.md` 的 Step 2 互動選單加入新選項
3. 無需修改 skill_writer.py 或 schema（採集器是獨立工具）

**介面約定（未正式文件化，從現有採集器歸納）：**
- `--setup`：初次設定（儲存 token/config 到 `~/.colleague-skill/{source}_config.json`）
- `--name`：目標人物姓名
- `--output-dir`：輸出目錄
- `--msg-limit`：訊息數量上限
- 輸出：`messages.txt` + `collection_summary.json`

## 6. SKILL.md 觸發詞擴展

**位置：** `SKILL.md:22-47`

觸發詞是在 SKILL.md 中以 Markdown 列表定義的純文字，不需要修改任何 Python 程式碼即可新增觸發詞。

## 7. 版本策略擴展點

**位置：** `tools/version_manager.py:MAX_VERSIONS`（第 21 行）

```python
MAX_VERSIONS = 10  # 最多保留 10 個版本
```

可調整版本保留數量，或擴展 `cleanup_old_versions()` 函式實作更複雜的版本策略（如按時間保留、按大小限制）。
