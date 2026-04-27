# 設定與環境分析

## 設定載入機制

本專案**沒有集中式設定檔**，設定分散在以下幾個層次：

### 1. 環境變數

| 環境變數 | 預設值 | 用途 |
|----------|--------|------|
| `DOT_SKILL_AUTO_INSTALL_CLAUDE` | `"1"` | 控制 `skill_writer.py --action create` 後是否自動安裝到 Claude Code。設為 `"0"` 停用。 |

位置：`skill_writer.py:main()` 第 547-550 行
```python
auto_install_default = os.environ.get("DOT_SKILL_AUTO_INSTALL_CLAUDE", "1") != "0"
install_claude_skill = (
    args.install_claude_skill or auto_install_default
) and not args.no_install_claude_skill
```

### 2. 採集器設定文件（~/.colleague-skill/）

各採集器初次設定時（`--setup`）會儲存設定到 home 目錄：

| 文件 | 用途 | 格式 |
|------|------|------|
| `~/.colleague-skill/slack_config.json` | Slack Bot Token | JSON `{"token": "xoxb-..."}` |
| `~/.colleague-skill/feishu_config.json` | ⚠️ 推測路徑（feishu_auto_collector.py --setup 儲存位置） | JSON |
| `~/.colleague-skill/dingtalk_config.json` | ⚠️ 推測路徑（dingtalk_auto_collector.py --setup 儲存位置） | JSON |

注意：Slack 設定路徑在 `INSTALL.md:249` 確認，其他採集器儲存路徑未在讀取的程式碼中確認（⚠️ 未驗證）。

### 3. CLI 參數（最高優先級覆蓋）

`skill_writer.py` 的 `--base-dir` 參數可覆蓋預設的 storage root：

```python
# skill_presets.py:258-262
def resolve_storage_root(character, base_dir_arg=None):
    if base_dir_arg:
        return Path(base_dir_arg).expanduser()  # CLI 覆蓋
    return canonical_storage_root(character)     # 預設：skills/{character}/
```

### 4. meta.json 中的 Skill 設定

每個生成的 Skill 的 `meta.json` 儲存了該 Skill 的設定，包含：

```json
{
  "schema_version": "3",
  "kind": "meta-skill",
  "character": "colleague",
  "research_profile": "standard",
  "classification": {
    "language": "zh-CN",
    "tags": ["甩鍋高手", "字節范"]
  },
  "lifecycle": {
    "status": "active",
    "created_at": "...",
    "updated_at": "...",
    "version": "v1"
  },
  "engine": {
    "name": "dot-skill",
    "merge_strategy": "compact",
    "quality_profile": "budget-friendly",
    "knowledge_dirs": ["docs", "messages", "emails"]
  }
}
```

## 設定優先級（由高到低）

```
CLI 參數（--base-dir, --character, --research-profile）
    ↓
環境變數（DOT_SKILL_AUTO_INSTALL_CLAUDE）
    ↓
meta.json 中的 Skill 設定（更新時讀取）
    ↓
CHARACTER_PRESETS 預設值（skill_presets.py）
    ↓
schema 預設值（skill_schema.py:enrich_skill_meta()）
```

## Storage Root 設定

Storage root 由 `character` 決定，也支援 legacy 路徑向後相容：

| Character | Canonical Root | Legacy Root |
|-----------|---------------|-------------|
| `colleague` | `skills/colleague` | `colleagues`（不同於 canonical，有 legacy fallback） |
| `relationship` | `skills/relationship` | `skills/relationship`（同 canonical，無 legacy） |
| `celebrity` | `skills/celebrity` | `skills/celebrity`（同 canonical，無 legacy） |

`resolve_existing_storage_root()` 函式（`skill_presets.py:265-287`）在讀取時會先試 canonical，若不存在再試 legacy，確保向後相容。

## Feature Flags

目前無正式 feature flag 機制，但有以下隱性開關：

| 開關 | 控制方式 | 預設行為 |
|------|---------|---------|
| Claude Code 自動安裝 | `DOT_SKILL_AUTO_INSTALL_CLAUDE=0` 或 `--no-install-claude-skill` | 自動安裝（ON） |
| Windows command shim | `--install-claude-command-shim` 或 `should_install_command_shim()` 函式 | 自動偵測 Windows |
| Playwright 依賴 | 執行時 try/except 動態 import | 可選 |
| pypinyin 依賴 | `skill_writer.py:slugify()` 第 117-125 行 try/except | 回退到 ASCII filter |

## Secrets 管理

本專案無加密儲存機制，所有 API token 明文儲存：
- Feishu token：推測儲存在 `~/.colleague-skill/feishu_config.json`（⚠️ 未驗證）
- Slack token：`~/.colleague-skill/slack_config.json`（明文 JSON）
- 飛書 App Secret 等敏感資訊由使用者自行保管，腳本執行時通過 `--setup` 讀入

⚠️ 安全建議：考慮使用作業系統 keychain 儲存 token，目前實作存在 secrets 洩露風險。

## Python 依賴設定

**位置：** `requirements.txt`

設計哲學：**選用最少強制依賴**，所有非核心功能設為可選：

```
requests>=2.28.0    # 必選（HTTP 呼叫）
pypinyin>=0.48.0    # 可選（中文→slug）
playwright>=1.40.0  # 可選（Feishu 瀏覽器採集 + 釘釘訊息）
slack-sdk>=3.27.0   # 可選（Slack 採集）
python-docx>=1.1.0  # 可選（Word 解析）
openpyxl>=3.1.0     # 可選（Excel 解析）
```

使用者可以只安裝 `requests` 即可完成基礎流程（手動貼上文字原材料），按需安裝其他可選依賴。
