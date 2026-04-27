# 核心邏輯分析

## 專案的「心臟」

dot-skill 的核心邏輯分散在三個層次：

1. **SKILL.md**：AI 的「主控腦」— 定義整個蒸餾流程的指令集
2. **skill_writer.py + skill_schema.py + skill_presets.py**：「執行腦」— 管理 artifact 生命週期
3. **Prompt 模板庫**：「知識腦」— 定義如何分析原材料、如何構建 Persona/Work

---

## 核心抽象 1：Character Family Preset 系統

**位置：** `tools/skill_presets.py`

這是整個多家族（colleague/relationship/celebrity）擴展能力的核心設計。

```python
# skill_presets.py:23-167
CHARACTER_PRESETS = {
    "colleague": { ... },     # 同事/同學/師長類
    "relationship": { ... },  # 情感/親情類
    "celebrity": { ... },     # 公眾人物/虛構角色類
}
```

每個 preset 定義：
- `prompt_bundle`：該 family 使用哪套 prompts
- `storage_root`：輸出到哪個目錄（`skills/colleague/`, `skills/relationship/`, `skills/celebrity/`）
- `knowledge_dirs`：知識庫目錄結構
- `is_real_person`, `is_public_figure`, `is_fictional`：語義屬性
- `command_aliases`：相容的舊指令名稱
- `skill_name_prefix`：生成 Skill 的命名前綴

celebrity 額外定義 `research_profiles`（budget-friendly vs budget-unfriendly），每個 profile 有獨立的：
- 最低品質門檻（`min_raw_notes`, `min_grounded_urls`...）
- `merge_strategy`（compact vs deep）
- `required_review_files`

**擴展模式：** 新增 character family 只需在 `CHARACTER_PRESETS` 加一個 key，無需修改核心流程。

---

## 核心抽象 2：Metadata Schema（`skill_schema.py`）

**位置：** `tools/skill_schema.py`

`enrich_skill_meta()` 函式（第 153-248 行）是元數據正規化的核心，負責：
- 向後相容：從舊格式（legacy `type` 欄位）升級到新格式（`character` 欄位）
- 填充 schema 中所有欄位的預設值
- 生成 `artifacts` dict（定義所有 artifact 的檔名和命令名）

生成的 meta 有多個分組（`lifecycle`, `generation`, `classification`, `source_context`, `engine`, `compat`），記錄了完整的溯源資訊。

`build_artifact_names()` 函式（第 103-123 行）實作命名策略：
```python
command_slug = slug.replace("_", "-")        # zhangsan → zhangsan
command_base = f"{character}-{command_slug}" # colleague-zhangsan
# 生成：/colleague-zhangsan, /colleague-zhangsan-work, /colleague-zhangsan-persona
```

---

## 核心抽象 3：Persona 六層結構

**位置：** `prompts/persona_builder.md` + `prompts/{family}/persona_builder.md`

所有 family 的 Persona 都採用**優先級層次結構**，Layer 0 優先級最高，任何情況下不得違背：

```
Layer 0 — 硬覆蓋層（手動標籤直接翻譯）
  最高優先級，定義「不能打破」的行為規則
  例：「被人質疑方案時，不解釋，反問對方判斷依據」

Layer 1 — 身份層
  你是誰、在哪裡工作、MBTI、文化背景

Layer 2 — 表達風格層（從原材料提取）
  口頭禪、句式、emoji 習慣、正式程度

Layer 3 — 決策與判斷層（從原材料提取）
  優先考量、推進/回避觸發

Layer 4 — 人際行為層（從原材料提取）
  對上/下/平級、壓力下

Layer 5 — Correction 記錄（滾動更新）
  每條記錄 {scene, wrong, correct}，即時生效
```

這套結構設計使 Persona 可以：
- 透過標籤快速創建基礎版本
- 透過原材料豐富 Layer 2-4
- 透過對話糾正持續精化 Layer 5

---

## 核心抽象 4：Skill Artifact 集

**位置：** `skill_writer.py:write_artifacts()` 第 197-223 行

每個生成的 Skill 都包含**完整的 artifact 集**：

| Artifact | 用途 | 使用者場景 |
|----------|------|-----------|
| `SKILL.md` | 合併版（完整 Persona + Work + 運行規則） | 主要使用場景，`/{character}-{slug}` |
| `work.md` | Work 原始內容（無 frontmatter） | 供 merger 做增量合併時讀取 |
| `persona.md` | Persona 原始內容（無 frontmatter） | 供 merger 做增量合併時讀取 |
| `work_skill.md` | Work-only Skill（帶 frontmatter） | `/{character}-{slug}-work` |
| `persona_skill.md` | Persona-only Skill（帶 frontmatter） | `/{character}-{slug}-persona` |
| `manifest.json` | 機器可讀元數據 | 安裝器、Gallery |
| `meta.json` | 完整人類+機器可讀元數據 | 版本管理、skill_writer 更新 |

**設計原因：** 分離 raw content（`work.md`/`persona.md`）和 rendered skill（`work_skill.md`/`SKILL.md`），使增量更新時只需 patch raw content 再重新 render，不需要複雜的 SKILL.md 解析。

---

## 核心算法：Markdown Patch 合併

**位置：** `skill_writer.py:merge_markdown_patch()` 第 264-299 行

```python
def merge_markdown_patch(existing_content: str, patch_content: str) -> str:
    """Replace matching level-2 markdown sections, otherwise append the patch."""
```

算法邏輯：
1. 用正則 `^##\s+.+$` 找出 patch 中所有 `##` 節標題
2. 對每個節，在 existing_content 中搜索相同標題
3. **找到**：用 patch 節內容完整替換現有節
4. **找不到**：將 patch 節附加到末尾
5. 若 patch 完全沒有 `##` 節：直接附加整個 patch

這使得增量更新不需要解析複雜的文件結構，只需要按節匹配即可。

---

## 核心算法：Research 品質驗證

**位置：** `tools/research/merge_research.py`

`summarize_research_files()` 函式（第 141-251 行）計算以下品質指標：
- URL 數量（`URL_PATTERN` 正則提取唯一 URL）
- Primary source 標記（搜索「first-person/primary source/一手/原始」）
- 潛在長引文行數（`count_potential_long_quote_lines()`，防版權風險）
- budget-unfriendly 額外：source weight 分佈（1-7 tier）、矛盾 bullet 數、推斷 bullet 數

這些指標作為「品質關卡」，防止 celebrity Skill 品質過低或存在版權風險。

---

## 核心模式：Correction Layer 更新

**位置：** `skill_writer.py:apply_correction()` 第 302-322 行

```python
def apply_correction(persona_content: str, correction: dict) -> str:
    """Append a normalized correction entry to persona content."""
    scene = correction.get("scene", "general")
    correction_line = f"\n- [{scene}] should not {correction['wrong']}; should {correction['correct']}"
```

每個 correction 以結構化格式記錄：`{scene, wrong, correct}`，附加到 `## Correction Log` 節下。Persona Skill 運行時，Layer 5 的 correction 規則優先於 Layer 2-4 的推斷規則。

---

## 核心模式：語言感知渲染

**位置：** `skill_writer.py:prefers_chinese()` 第 143-145 行

```python
def prefers_chinese(meta: dict) -> bool:
    return language_code(meta).startswith("zh")
```

根據 `meta.classification.language` 欄位，選擇用中文或英文模板渲染 `SKILL.md`。兩套模板（`SKILL_MD_TEMPLATE_ZH` / `SKILL_MD_TEMPLATE_EN`）第 37-108 行定義。
