# 資料流分析

## 代表性 Use Case：建立一個同事 Skill

以「從飛書自動採集並建立一個 colleague Skill」為例，完整追蹤資料流。

### 全流程概覽

```
使用者輸入
    ↓
[Agent] 讀取 SKILL.md（系統提示注入）
    ↓
[互動] Step 0-1：確認 character = colleague，錄入基礎資訊
    ↓
[工具呼叫] Step 2：飛書自動採集
    → python3 tools/feishu_auto_collector.py --name "{name}" --output-dir ./knowledge/{slug}
    → 輸出：knowledge/{slug}/messages.txt, docs.txt, collection_summary.json
    ↓
[Agent] Step 3：讀取 prompts/work_analyzer.md + prompts/persona_analyzer.md
    → 分析原材料（LLM context 內處理）
    → 輸出：work 分析結論 + persona 分析結論
    ↓
[Agent] Step 4：讀取 prompts/work_builder.md + prompts/persona_builder.md
    → 生成 draft work 內容 + persona 內容
    → 展示預覽，等待確認
    ↓
[工具呼叫] Step 5：寫出 artifact
    → Write /tmp/dot_skill_{slug}_meta.json
    → Write /tmp/dot_skill_{slug}_work.md
    → Write /tmp/dot_skill_{slug}_persona.md
    → python3 tools/skill_writer.py --action create --character colleague --slug {slug} ...
    ↓
[輸出] skills/colleague/{slug}/ 完整目錄
```

### 層次分析

#### 1. 路由層（Routing）

路由由 SKILL.md 中的 Step 0 決定：

```markdown
# SKILL.md:82-98
Step 0：確認 character family
- 如果用戶使用 /dot-skill，先確認本次要蒸餾的是哪一類：
  1. colleague
  2. relationship
  3. celebrity
```

Character 確認後，`skill_presets.py:normalize_character()` 函式（第 181-187 行）決定 prompt_bundle 路徑。

#### 2. 資料採集層（Data Ingestion）

**路徑：** `SKILL.md:Step 2` → 工具呼叫

支援的採集管道：

| 管道 | 工具 | 輸出格式 |
|------|------|---------|
| 飛書 API（群聊） | `feishu_auto_collector.py` | `messages.txt` + `docs.txt` + `collection_summary.json` |
| 飛書 API（私聊） | `feishu_auto_collector.py --p2p-chat-id` | 同上 |
| 飛書瀏覽器 | `feishu_browser.py` | 純文字文件 |
| 飛書 MCP | `feishu_mcp_client.py` | 純文字文件 |
| 釘釘 API + 瀏覽器 | `dingtalk_auto_collector.py` | `docs.txt` + `bitables.txt` + `messages.txt` |
| Slack API | `slack_auto_collector.py` | `messages.txt` + `collection_summary.json` |
| 飛書 JSON 導出 | `feishu_parser.py` | 純文字 |
| 郵件 | `email_parser.py` | 純文字 |
| PDF/圖片/MD | Read 工具（原生支援） | 直接讀取 |

#### 3. 分析層（Analysis / Transformation）

**Work 分析（`prompts/work_analyzer.md`）：**

提取 5 個維度：
- ① 負責系統/業務（服務、模組、文件、職責邊界）
- ② 技術規範與偏好（程式碼風格、接口設計、架構偏好）
- ③ 工作流程（需求處理、技術方案、Code Review、線上問題處理）
- ④ 輸出格式偏好（文件結構、回覆格式）
- ⑤ 知識庫（常引用的技術方案、積累的經驗結論）

**Persona 分析（`prompts/persona_analyzer.md`）：**

提取 4 個維度：
- ① 表達風格（口頭禪、高頻詞、句式特徵、emoji 習慣）
- ② 決策模式（優先考量、推進觸發、回避觸發）
- ③ 人際行為（對上級/下級/平級/壓力下）
- ④ 邊界與雷區

此層純在 LLM context 內處理，無外部 I/O。

#### 4. 生成層（Persistence / Build）

**Work Builder（`prompts/work_builder.md`）：** 生成 `work.md` 草稿。

**Persona Builder（`prompts/persona_builder.md`）：** 生成 `persona.md` 草稿，包含 6 層結構：
- Layer 0：硬覆蓋層（最高優先級行為規則）
- Layer 1：身份層
- Layer 2：表達風格層
- Layer 3：決策與判斷層
- Layer 4：人際行為層
- Layer 5（Correction 記錄）

#### 5. 持久化層（Storage）

**路徑：** `skill_writer.py:create_skill()` → `write_artifacts()` 函式

寫出文件（`skill_schema.py:PRIMARY_ARTIFACTS` 第 20-27 行）：

```python
PRIMARY_ARTIFACTS = (
    "SKILL.md",
    "work.md",
    "persona.md",
    "work_skill.md",
    "persona_skill.md",
    "manifest.json",
)
```

加上：`meta.json`（含完整元數據）

**目錄結構：**
```
skills/colleague/{slug}/
├── SKILL.md         ← 合併版（Work + Persona + 運行規則）
├── work.md          ← Work 原始內容
├── persona.md       ← Persona 原始內容
├── work_skill.md    ← 可獨立調用的 Work Skill（帶 frontmatter）
├── persona_skill.md ← 可獨立調用的 Persona Skill（帶 frontmatter）
├── manifest.json    ← 機器可讀元數據（安裝/Gallery 用）
├── meta.json        ← 完整元數據（含版本、標籤、生成設定）
├── versions/        ← 歷史版本存檔
└── knowledge/       ← 原始材料（不含在 PRIMARY_ARTIFACTS 中）
    ├── docs/
    ├── messages/
    └── emails/
```

#### 6. 回應層（Response）

LLM 告知使用者：
- 文件所在路徑（按 character family 正確返回）
- 調用命令：`/{character}-{slug}`（例：`/colleague-zhangsan`）
- Work-only：`/{character}-{slug}-work`
- Persona-only：`/{character}-{slug}-persona`

### 更新資料流（進化模式）

```
使用者觸發更新（追加文件/對話糾正）
    ↓
[Agent] 讀取現有 work.md + persona.md
    ↓
[分析] 使用 merger prompt 分析增量
    ↓
[備份] python3 tools/version_manager.py --action backup ...
    ↓
[寫出 patch] /tmp/dot_skill_{slug}_work_patch.md
    ↓
[更新] python3 tools/skill_writer.py --action update --work-patch ...
    ↓
[合併邏輯] skill_writer.py:merge_markdown_patch()
    → 按 ## 節標題匹配，找到則替換，找不到則附加
    ↓
[重新生成] write_artifacts() 更新所有 artifact
    ↓
[版本遞增] v1 → v2 → ...
```

### Celebrity 研究資料流（額外子流程）

```
celebrity family 選擇
    ↓
Step 3 前，celebrity 研究子流程：
    ↓
[建立目錄] mkdir knowledge/research/raw + merged
    ↓
[六維度研究]（budget-friendly：3+ 檔，budget-unfriendly：6 檔）
    → LLM 搜尋並整理研究筆記
    → Write 到 knowledge/research/raw/01_core_profile.md 等
    ↓
[合併] python3 tools/research/merge_research.py "{skill_dir}"
    → 輸出：knowledge/research/merged/summary.md
    ↓
[品質門檻] merge_research.py 輸出統計指標
    → Files scanned >= 3（budget-friendly）
    → Unique URLs >= 2
    → Potential long quote lines = 0
    ↓
[品質關卡] Phase 1.5：展示結構化採集摘要，等待確認
    ↓
budget-unfriendly 額外流程：
    → 生成 research_audit.md（PASS/FAIL）
    → 生成 synthesis.md（mental models triple-gate）
    → 生成 validation.md（known-answer check）
```
