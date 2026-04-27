# dot-skill 開發者上手指南

> 最後更新：2026-04-27

---

## 目錄

1. [Prerequisites 與環境建置](#1-prerequisites-與環境建置)
2. [本地開發 Workflow](#2-本地開發-workflow)
3. [測試策略與執行方式](#3-測試策略與執行方式)
4. [Debugging 技巧](#4-debugging-技巧)
5. [Contribution Workflow](#5-contribution-workflow)
6. [Windows 特殊注意](#6-windows-特殊注意)
7. [開發者 Workflow 示意圖](#7-開發者-workflow-示意圖)

---

## 1. Prerequisites 與環境建置

### Python 版本確認

```bash
python3 --version   # 需要 3.9 或以上
```

CI 矩陣測試 Python `3.9` 與 `3.11`（見 `.github/workflows/ci.yml`）。

### 必選依賴

```bash
pip3 install requests       # HTTP API 呼叫（Feishu、Slack 等），>=2.28.0
```

`requests` 是唯一的強制依賴，其他套件全部為可選。

### 可選依賴與安裝場景

| 套件 | 版本要求 | 安裝時機 |
|------|---------|---------|
| `pypinyin` | >=0.48.0 | 需要將中文姓名自動轉為拼音 slug 時（否則 `slugify()` 會回退到 ASCII filter） |
| `playwright` | >=1.40.0 | 使用 `feishu_browser.py` 做瀏覽器登入態採集，或釘釘訊息採集 |
| `slack-sdk` | >=3.27.0 | 使用 `slack_auto_collector.py` 自動從 Slack 採集訊息 |
| `python-docx` | >=1.1.0 | 需要解析 Word `.docx` 文件作為原始材料 |
| `openpyxl` | >=3.1.0 | 需要解析 Excel `.xlsx` 文件作為原始材料 |

```bash
# 推薦一次性安裝全部依賴（開發者）
pip3 install -r requirements.txt

# Playwright 需要額外安裝瀏覽器引擎
pip3 install playwright
playwright install chromium   # 僅需 chromium，不需完整 Chrome

# Feishu MCP 方案（Node.js 16+）
npm install -g feishu-mcp
```

各依賴的 `import` 均以 `try/except` 包裹，缺失時優雅降級，不會直接崩潰。

### 各 Agent 宿主的安裝路徑

| 宿主 | Meta-skill 路徑 | 生成角色 Skill 路徑 |
|------|---------------|-----------------|
| Claude Code（專案） | `.claude/skills/dot-skill/` | `~/.claude/skills/{character}-{slug}/` |
| Claude Code（全域） | `~/.claude/skills/dot-skill/` | 同上 |
| Hermes | `~/.hermes/skills/dot-skill/`（⚠️ 由安裝器決定） | — |
| OpenClaw | `~/.openclaw/workspace/skills/dot-skill/` | `~/.openclaw/workspace/skills/{character}-{slug}/` |
| Codex | `~/.codex/skills/dot-skill/` | `~/.codex/skills/{character}-{slug}/` |

---

## 2. 本地開發 Workflow

從 clone 到第一次成功執行 `/dot-skill`，共 5 個步驟。

### Step 1：Clone Repo

```bash
git clone https://github.com/titanwings/colleague-skill.git
cd colleague-skill
```

### Step 2：選擇安裝到哪個 Agent 宿主

**A. Claude Code（推薦，安裝到當前專案）**

```bash
# 必須在 git 倉庫根目錄執行
cd $(git rev-parse --show-toplevel)
mkdir -p .claude/skills
git clone https://github.com/titanwings/colleague-skill .claude/skills/dot-skill
```

**B. Claude Code（全域，所有專案都能用）**

```bash
git clone https://github.com/titanwings/colleague-skill ~/.claude/skills/dot-skill
```

**C. Hermes**

```bash
# clone 後，再執行安裝器同步
python3 tools/install_hermes_skill.py --force
hermes skills list | grep dot-skill
```

**D. OpenClaw**

```bash
python3 tools/install_openclaw_skill.py --force
# 或直接 clone
git clone https://github.com/titanwings/colleague-skill ~/.openclaw/workspace/skills/dot-skill
```

**E. Codex**

```bash
python3 tools/install_codex_skill.py --force
# 或直接 clone
git clone https://github.com/titanwings/colleague-skill ~/.codex/skills/dot-skill
```

### Step 3：安裝 Python 依賴

```bash
pip3 install -r requirements.txt
```

最小可執行環境只需 `pip3 install requests`，其他可選依賴按需安裝。

### Step 4：初次設定採集器（若需要）

**Feishu 自動採集（API 方案）：**

```bash
python3 tools/feishu_auto_collector.py --setup
# 按提示輸入飛書開放平台 App ID 和 App Secret
# 設定儲存到 ~/.colleague-skill/feishu_config.json
```

**Slack 採集：**

```bash
pip3 install slack-sdk
python3 tools/slack_auto_collector.py --setup
# 輸入 Bot User OAuth Token（xoxb-...）
# 設定儲存到 ~/.colleague-skill/slack_config.json（INSTALL.md:249 確認）
```

**釘釘採集：**

```bash
python3 tools/dingtalk_auto_collector.py --setup
# 首次加 --show-browser 完成釘釘登入
```

**飛書瀏覽器方案（無 App 權限時）：**

```bash
pip3 install playwright && playwright install chromium
python3 tools/feishu_browser.py \
  --url "https://xxx.feishu.cn/wiki/xxx" \
  --show-browser   # 首次需要登入，之後自動複用登入態
```

若只做手動貼文字原材料，可跳過本步驟。

### Step 5：啟動並測試

在支援 slash command 的宿主中輸入：

```
/dot-skill
```

或用 CLI 快速驗證：

```bash
python3 tools/skill_writer.py --action list
python3 tools/install_hermes_skill.py --dry-run
python3 tools/research/quality_check.py --help
```

---

## 3. 測試策略與執行方式

### 執行全部測試

```bash
python -m unittest discover -s tests -p 'test_*.py' -v
```

在提交 PR 前，也需要確認 Python 編譯無誤：

```bash
python -m compileall tools/
```

### 靜態分析

```bash
pip install ruff
ruff check tools/
```

目前 ruff 為**非阻塞**（`|| true`），CI 不會因 lint 失敗而擋 PR，但本地建議修復。

### CI 設定（`.github/workflows/ci.yml`）

- **test job**：在 `ubuntu-latest` 上分別跑 Python `3.9` 與 `3.11`
  1. `pip install -r requirements.txt`
  2. `python -m compileall -q tools`（編譯檢查）
  3. `python -m unittest discover -s tests -p 'test_*.py' -v`
- **lint job**：獨立 job，只跑 `ruff check tools/ || true`（非阻塞）

### tests/ 目錄下 7 個測試文件

| 測試文件 | 測試範圍 |
|---------|---------|
| `test_skill_writer.py` | `skill_writer.py` 核心邏輯：create/update/rollback、三種 character family 的 meta.json 與 manifest.json 結構、中英文 Chrome 輸出、`budget-unfriendly` profile 設定、Correction 合併去重 |
| `test_cli_lifecycle.py` | 完整 CLI 生命週期（end-to-end subprocess 測試）：create → list → update → rollback；三種 character 的完整流程；celebrity research toolchain（srt_to_transcript、merge_research、quality_check）；多宿主安裝器 CLI flag |
| `test_install_claude_generated_skill.py` | Claude Code 角色 Skill 安裝：`install_generated_skill()` 寫入目錄、`.dot-skill-install.json` metadata、Windows command shim 寫入、`should_install_command_shim()` 平台偵測邏輯 |
| `test_install_hermes_skill.py` | Hermes meta-skill 安裝器：`install_skill()` 複製 repo layout、`--dry-run` 不寫入、dry-run 允許 destination 已存在 |
| `test_install_openclaw_and_codex.py` | OpenClaw / Codex meta-skill 安裝器 + 角色 Skill 安裝器：四個安裝器的複製行為與 `.dot-skill-install.json` metadata |
| `test_research_tools.py` | Celebrity research toolchain：`srt_to_transcript`（清理時間戳、去重）、`merge_research`（summary.md 格式）、`quality_check`（budget-friendly 與 budget-unfriendly 品質門檻、generic URL 拒絕） |
| `test_skill_entrypoint_docs.py` | 文件一致性驗證：SKILL.md 包含正確的 `/dot-skill` 入口、README.md 與 INSTALL.md 路徑一致性、`skills/colleague/` 範例目錄存在、7 種語言 README 包含 `/dot-skill` 與 research toolchain |

---

## 4. Debugging 技巧

### `--dry-run` Flag（安裝器）

所有宿主安裝器均支援 `--dry-run`，不實際寫入任何檔案：

```bash
python3 tools/install_hermes_skill.py --dry-run
python3 tools/install_openclaw_skill.py --dry-run
python3 tools/install_codex_skill.py --dry-run
```

用於在正式安裝前確認目標路徑正確。

### 確認 Skill 是否正確寫出

```bash
# 列出 colleague 類 Skills
python3 tools/skill_writer.py --action list --base-dir ./skills/colleague

# 列出 celebrity 類 Skills
python3 tools/skill_writer.py --action list --base-dir ./skills/celebrity
```

輸出應包含各 slug 與 `Character: {character}` 欄位。

### 停用 Claude Code 自動安裝

`skill_writer.py --action create` 預設在創建後自動安裝到 Claude Code（`DOT_SKILL_AUTO_INSTALL_CLAUDE` 預設為 `"1"`）。在測試或 CI 環境中應關閉：

```bash
export DOT_SKILL_AUTO_INSTALL_CLAUDE=0
python3 tools/skill_writer.py --action create ...
# 或
python3 tools/skill_writer.py --action create ... --no-install-claude-skill
```

對應程式碼位置：`tools/skill_writer.py` 第 547–550 行。

### 採集器常見報錯與解決

| 報錯訊息 | 原因 | 解決方式 |
|---------|------|---------|
| `missing_scope: channels:history` | Slack Bot Token 缺少 scope | 回到 api.slack.com 補充 scope 後重新安裝 App |
| `invalid_auth` | Token 無效 | 重新執行 `slack_auto_collector.py --setup` |
| `not_in_channel` | Bot 未加入頻道 | 在 Slack 中 `/invite @bot-name` |
| HTTP 429 | 請求過頻 | 腳本已有自動 retry，無需手動干預 |
| `ModuleNotFoundError: playwright` | playwright 未安裝 | `pip3 install playwright && playwright install chromium` |
| `ModuleNotFoundError: pypinyin` | pypinyin 未安裝 | `pip3 install pypinyin`（或接受 ASCII fallback） |
| 消息只有 90 天 | Slack 免費版限制 | 升級 Workspace 或手動補充截圖 |

### 驗證生成的 Skill 結構正確性

```bash
# celebrity 品質檢查（輸出應含 OVERALL PASS）
python3 tools/research/quality_check.py skills/celebrity/{slug}/SKILL.md

# 確認必要段落與 schema
grep -E "^## PART (A|B)" skills/colleague/{slug}/SKILL.md
python3 -c "import json; m=json.load(open('skills/colleague/{slug}/meta.json')); print(m['schema_version'], m['kind'])"
# 應輸出：## PART A/B ... / 3 meta-skill
```

---

## 5. Contribution Workflow

### Branch 命名規則

基於 `CONTRIBUTING.md` 第 36–41 行：

| 類型 | 格式 |
|------|------|
| 新功能 | `feat/<short-name>` |
| Bug 修復 | `fix/<short-name>` |
| 文件 | `docs/<short-name>` |
| 工具 / infra | `chore/<short-name>` |

### PR 規範

- Fork repo 後從 `main` 建立 branch
- 每個 PR 只處理一個關注點（one concern per PR）
- PR 前先在本地通過編譯與測試：
  ```bash
  python -m compileall tools/
  python -m unittest discover -s tests -p 'test_*.py' -v
  ```
- PR target branch：`main`
- CI 必須全部通過才能合併
- Commit message 遵循 [Conventional Commits](https://www.conventionalcommits.org/)，subject 在 72 字元以內

Commit message 格式範例：
```
feat: add Notion auto-collector
fix: handle 429 rate limit in feishu_parser
docs: translate INSTALL to Korean
test: cover skill_writer rollback edge cases
```

### 新增 Character Family 的步驟

根據 `.trace/_context/extensions.md` 第 4–26 行與 `tools/skill_presets.py`：

1. **在 `CHARACTER_PRESETS` 新增 preset**（`tools/skill_presets.py` 第 23 行）：

   ```python
   CHARACTER_PRESETS["self"] = {
       "character": "self",
       "prompt_bundle": {
           "preset": "dot.self.v1",
           "intake": "prompts/self/intake.md",
           "persona_analyzer": "prompts/self/persona_analyzer.md",
           "persona_builder": "prompts/self/persona_builder.md",
           # ...
       },
   }
   ```

2. **在 `prompts/` 建立對應目錄**，放入必要的 prompt 文件（`intake.md`、`persona_analyzer.md`、`persona_builder.md`、`merger.md`）。

3. **`skills/` 目錄無需手動建立**，`skill_writer.py` 會自動創建 `skills/{character}/{slug}/`。

4. **可選：在 `CHARACTER_ALIASES` 加入別名**（`skill_presets.py` 第 170–178 行）：

   ```python
   CHARACTER_ALIASES["myself"] = "self"
   ```

5. **新增對應測試**，涵蓋 preset metadata、prompt bundle 路徑存在性。

### 新增採集器的步驟

根據 `.trace/_context/extensions.md` 第 90–100 行：

1. **建立 `tools/{source}_auto_collector.py`**，實作以下 CLI 介面：
   - `--setup`：初次設定（token 儲存到 `~/.colleague-skill/{source}_config.json`，權限建議設為 `0600`）
   - `--name`：目標人物姓名
   - `--output-dir`：輸出目錄
   - `--msg-limit`：訊息數量上限（可選）
   - 輸出：`messages.txt` + `collection_summary.json`

2. **在 `SKILL.md` 的 Step 2 互動選單加入新選項**（採集器是獨立工具，無需修改 `skill_writer.py` 或 schema）。

3. **新增測試**，至少涵蓋：auth 模式、rate-limit/retry 行為（mock HTTP）、輸出格式與現有採集器一致性。CI 中禁止打真實 API，請用 `unittest.mock` 或 `responses` 庫。

---

## 6. Windows 特殊注意

### Claude Code Windows 的 Command Shim 安裝

Windows 上 Claude Code 目前存在 skill 發現問題，需要額外寫入 command shim 到 `~/.claude/commands/{character}-{slug}.md`：

```bash
# 生成角色 Skill 時同時安裝 command shim
python3 tools/install_claude_generated_skill.py \
  --skill-dir skills/celebrity/zhou_qimo \
  --install-claude-command-shim \
  --force
```

或在 `skill_writer.py` 中加 flag：

```bash
python3 tools/skill_writer.py --action create \
  --character celebrity --slug zhou_qimo --name "周奇墨" \
  --meta meta.json --work work.md --persona persona.md \
  --install-claude-skill \
  --install-claude-command-shim   # 僅 Windows 需要
```

### `should_install_command_shim()` 函式行為

位置：`tools/install_claude_generated_skill.py` 第 27–30 行：

```python
def should_install_command_shim(system_name: str | None = None) -> bool:
    """Return whether a slash-command shim should be installed."""
    current = (system_name or platform.system()).lower()
    return current.startswith("win")
```

- 傳入 `"Windows"` → 回傳 `True`，安裝 command shim
- 傳入 `"Darwin"` 或 `"Linux"` → 回傳 `False`，不安裝
- 不傳入 → 自動呼叫 `platform.system()` 偵測當前作業系統

測試覆蓋位置：`tests/test_install_claude_generated_skill.py` 第 92–94 行。

---

## 7. 開發者 Workflow 示意圖

```mermaid
flowchart TD
    A([Fork & Clone]) --> B[選擇 Agent 宿主]

    B --> C1[Claude Code\ngit clone → .claude/skills/dot-skill]
    B --> C2[Hermes\ninstall_hermes_skill.py --force]
    B --> C3[OpenClaw / Codex\ninstall_openclaw/codex_skill.py --force]

    C1 & C2 & C3 --> D[pip3 install -r requirements.txt]

    D --> E{需要採集器？}
    E -- 是 --> F[執行 --setup\n儲存 token 到\n~/.colleague-skill/]
    E -- 否 --> G

    F --> G[啟動 /dot-skill\n在 Agent 宿主中測試]

    G --> H[開發修改]

    H --> I{修改類型}
    I -- 新 character --> J[skill_presets.py\n+ prompts/{family}/\n+ 測試]
    I -- 新採集器 --> K[tools/{source}_auto_collector.py\n+ SKILL.md 選單\n+ 測試]
    I -- Prompt 調整 --> L[prompts/**/*.md]
    I -- Bug fix --> M[tools/*.py\n+ 測試]

    J & K & L & M --> N[本地驗證\npython -m compileall tools/\npython -m unittest discover -s tests]

    N -- 失敗 --> H
    N -- 通過 --> O[ruff check tools/\n非阻塞，盡量修復]

    O --> P[feat/fix/docs/chore\n分支 + Conventional Commit]
    P --> Q[開 PR → main\nCI 必須全部 green]
    Q --> R{Review 通過？}
    R -- 需修改 --> H
    R -- 通過 --> S([Merge to main])
```

---

## 附錄：快速參考

### 常用 CLI 指令速查

```bash
# 創建 Skill
python3 tools/skill_writer.py --action create \
  --character colleague --slug zhangsan --name "張三" \
  --meta meta.json --work work.md --persona persona.md

# 更新 Skill（追加 work patch）
python3 tools/skill_writer.py --action update \
  --character colleague --slug zhangsan \
  --work-patch /tmp/patch.md

# 列出所有 colleague Skills
python3 tools/skill_writer.py --action list --base-dir ./skills/colleague

# 版本回滾
python3 tools/version_manager.py --action rollback \
  --character colleague --slug zhangsan --version v1

# 執行全部測試
python -m unittest discover -s tests -p 'test_*.py' -v
```

### Storage Root 對照表

| Character | Canonical Root | Legacy Root |
|-----------|---------------|-------------|
| `colleague` | `skills/colleague` | `colleagues`（有 legacy fallback） |
| `relationship` | `skills/relationship` | 同 canonical |
| `celebrity` | `skills/celebrity` | 同 canonical |

`resolve_existing_storage_root()` 位置：`tools/skill_presets.py` 第 265–287 行。
