# dot-skill 探索紀錄與待解問題

> **產出日期：** 2026-04-27
> **偵察範圍：** `/home/user/colleague-skill`（125 個檔案）
> **文件狀態：** 初版，部分資訊標注 ⚠️ 未驗證

---

## 1. 既有文件與程式碼落差清單

以下落差來自 `recon.md` 的落差分析，已逐一確認位置。

| # | 文件說 | 程式碼實際 | 文件位置 | 程式碼位置 |
|---|--------|-----------|----------|-----------|
| D-01 | 主入口命令為 `/create-colleague` | 實際為 `/dot-skill`（升級後） | `docs/PRD.md` | `SKILL.md:37` |
| D-02 | 輸出目錄為 `~/.openclaw/workspace/skills/colleagues/` | 實際為 `./skills/colleague/` | `docs/PRD.md:七` | `tools/skill_presets.py:35` |
| D-03 | Persona 設計為「5 層（Layer 0-4）」 | 實際為 6 段（Layer 0-4 + Correction 記錄） | `docs/PRD.md:5.2` | `prompts/persona_builder.md` |
| D-04 | P0 功能標記為 `[ ]`（未完成） | 實際已完整實作並在 CI 中通過測試 | `docs/PRD.md:九` | `tests/test_skill_writer.py` |
| D-05 | 最多 10 個版本（文字描述） | `MAX_VERSIONS = 10`（已實作常數） | `docs/PRD.md:6.3` | `tools/version_manager.py:21` |
| D-06 | `feishu_config.json` 儲存路徑在 `configuration.md` 標注為「⚠️ 推測路徑」 | 程式碼中確認為 `~/.colleague-skill/feishu_config.json` | `.trace/_context/configuration.md:28` | `tools/feishu_auto_collector.py:59` |
| D-07 | `dingtalk_config.json` 儲存路徑同樣標注為「⚠️ 推測路徑」 | 程式碼中確認為 `~/.colleague-skill/dingtalk_config.json` | `.trace/_context/configuration.md:29` | `tools/dingtalk_auto_collector.py:45` |

**說明：** D-01 至 D-05 屬於文件未隨程式碼升級而更新的歷史遺留問題。D-06、D-07 為本次偵察過程中已完成自我修正的項目。

---

## 2. 已知技術債與改進點

### 2.1 Secrets 明文儲存（嚴重程度：中）

所有 API token 以明文 JSON 形式儲存於 home 目錄：

- `~/.colleague-skill/slack_config.json` → `{"token": "xoxb-..."}` — 確認於 `INSTALL.md:249`
- `~/.colleague-skill/feishu_config.json` → 包含 App ID + App Secret — 確認於 `tools/feishu_auto_collector.py:59`
- `~/.colleague-skill/dingtalk_config.json` → 包含 AppKey + AppSecret — 確認於 `tools/dingtalk_auto_collector.py:45`

目前無任何加密或 OS keychain 整合。若 home 目錄遭未授權存取，所有平台憑證將直接洩露。

### 2.2 ruff lint 為非阻塞（嚴重程度：低）

CI 的 lint 步驟設定：

```yaml
# .github/workflows/ci.yml（Ruff job）
- name: Run ruff (non-blocking for now)
  run: ruff check tools/ || true
```

`|| true` 使 ruff 報告的所有 lint 錯誤均不會導致 CI 失敗，程式碼品質問題可在不被注意的情況下合入主分支。

### 2.3 Celebrity research 工具鏈外部依賴未明文說明（嚴重程度：低）

`tools/research/download_subtitles.sh` 在執行時會硬性檢查 `yt-dlp`：

```bash
# tools/research/download_subtitles.sh:9-10
if ! command -v yt-dlp >/dev/null 2>&1; then
  echo "error: yt-dlp is required" >&2
```

`tools/research/transcribe_audio.py` 依賴以下三個 backend（優先順序）：
1. `faster-whisper`（本地，Apple Silicon 優先）
2. `openai-whisper`（本地 fallback）
3. OpenAI Whisper API（需設定 `OPENAI_API_KEY`）

這些工具未列入 `requirements.txt`，也未在 `INSTALL.md` 的 celebrity 安裝流程中說明。

### 2.4 feishu_mcp_client.py 的 Node.js 依賴未明文說明（嚴重程度：低）

MCP 客戶端依賴 `feishu-mcp`（npm 套件）且需要 Node.js 16+，但 `requirements.txt` 為純 Python，此依賴無法被自動安裝。

### 2.5 openarena-claim.txt 用途澄清

該檔案內容為：

```
OpenArena owner claim verification for titanwings/colleague-skill: AA5D7DAF373B45F884E4
```

**結論：** 這是 OpenArena（類似 OpenGraph 的平台）的 repo 擁有者驗證憑證文件，類似 GitHub Pages 的 `CNAME` 文件。對專案功能無影響，屬於平台聲明檔案。

---

## 3. 程式碼中的 TODO / FIXME / HACK

使用 `grep -rn "TODO\|FIXME\|HACK\|WORKAROUND"` 對整個 codebase 搜尋（排除 `.git/`），**結果：零命中**。

```
搜尋路徑：/home/user/colleague-skill/tools/
          /home/user/colleague-skill/prompts/
          /home/user/colleague-skill/tests/
          /home/user/colleague-skill/（全域）
結果：無任何 TODO / FIXME / HACK / WORKAROUND 標記
```

這可能表示：
- 開發過程中沒有使用這類標記的習慣；或
- 已完成的 TODO 在合入前被移除

---

## 4. 未解答的問題（需向維護者確認）

| # | 問題 | 影響範圍 | 優先級 |
|---|------|---------|--------|
| Q-01 | `feishu_browser.py` 的完整實作邏輯為何？Playwright 的 context 複用策略？ | 飛書瀏覽器採集流程 | 高 |
| Q-02 | `feishu_mcp_client.py` 是否要求飛書官方 MCP 伺服器在本地運行？啟動方式？ | MCP 整合流程 | 高 |
| Q-03 | `dingtalk_auto_collector.py` 瀏覽器採集的 selector 維護策略？釘釘 DOM 更新後如何應對？ | 釘釘訊息採集穩定性 | 中 |
| Q-04 | `transcribe_audio.py` 的 OpenAI Whisper API backend 是否需要額外設定（除 `OPENAI_API_KEY`）？ | Celebrity 音訊轉寫 | 中 |
| Q-05 | `celebrity` 的 `budget-unfriendly` 模式是否有完整的端對端使用教學？六個 prompt 的完整流程？ | Celebrity 深度研究 | 中 |
| Q-06 | `merge_strategy: "deep"`（budget-unfriendly 專屬）和 `"compact"` 的具體差異為何？ | Skill 合併邏輯 | 低 |
| Q-07 | `skills/` 目錄被 `.gitignore` 排除，範例 Skill（`example_zhangsan` 等）如何發布給用戶參考？ | 文件與教學 | 低 |
| Q-08 | Feishu user_access_token 2 小時到期後，refresh_token 刷新流程在哪裡實作？ | 飛書私聊採集可靠性 | 低 |

---

## 5. 需要更深入調查的區域

### 5.1 測試覆蓋盲點

當前有 7 個測試檔案，但以下功能**確認缺乏測試覆蓋**：

| 模組 | 測試狀態 | 說明 |
|------|---------|------|
| `feishu_auto_collector.py` | 無直接測試 | HTTP 採集邏輯需要 Mock API |
| `feishu_browser.py` | 無直接測試 | Playwright 整合測試複雜 |
| `feishu_mcp_client.py` | 無直接測試 | 需 MCP 伺服器 |
| `dingtalk_auto_collector.py` | 無直接測試 | 需真實 AppKey |
| `slack_auto_collector.py` | 無直接測試 | 需 Bot Token |
| `email_parser.py` | 無直接測試 | 需範例 .eml 文件 |
| `tools/research/*.py` | 僅有 `test_research_tools.py` | 覆蓋範圍未確認 |

### 5.2 未讀取完整實作的模組

以下模組的完整原始碼在本次偵察中未逐行讀取：

- `tools/feishu_browser.py` — Playwright context 管理策略未確認
- `tools/feishu_mcp_client.py` — MCP 協議交互細節未確認
- `tools/dingtalk_auto_collector.py` — 瀏覽器採集的 DOM selector 未確認

### 5.3 Celebrity budget-unfriendly 完整流程

`budget-unfriendly` 模式定義了六個獨立 prompt 文件（research → audit → synthesis → validation → persona_analyzer → persona_builder），對應 `skill_presets.py:126-143` 的配置，但完整的六步驟執行流程、品質門檻數值（`min_raw_notes`、`min_grounded_urls` 等具體數值）未在本次偵察中確認。

---

## 6. Web 搜尋發現摘要

資料來源：`.trace/_context/web_findings.md`

### 6.1 外部資源索引

| 資源 | 連結 | 關鍵 Takeaway |
|------|------|--------------|
| GitHub Repo | https://github.com/titanwings/colleague-skill | 截至 2026-04-19 達 15k Stars，社群採用廣泛 |
| 社群 Gallery | https://titanwings.github.io/colleague-skill-site/ | 100+ 社群貢獻 Skill，可作為學習範例 |
| AgentSkills 標準 | https://agentskills.io | 本 repo 遵循此開放標準格式 |
| Discord 社群 | https://discord.gg/NVX66RxWZv | 即時技術支援與討論 |
| Karpathy Skill 案例 | https://github.com/alchaincyf/karpathy-skill | 社群 celebrity Skill 的最佳實踐參考 |
| 技術論文 | `colleague_skill.pdf`（repo 根目錄） | Work Skill + Persona 雙層架構的學術依據 |

### 6.2 關鍵 Takeaway

1. **品牌升級**：2026-04-13 宣布從 `colleague-skill` 升級為 `dot-skill`，擴展定位從「蒸餾同事」到「蒸餾任何人」
2. **算力支持**：Shanghai AI Lab + AI Safety Center 提供算力，非純個人專案
3. **已知平台限制**：
   - Slack 免費版：90 天訊息記錄上限
   - 釘釘：訊息 API 不支援歷史記錄，強制切換瀏覽器採集
4. **Roadmap 方向**（對評估技術債優先級有參考價值）：
   - Phase 2：`/create-skill` 萬用入口（文件更新需求緊迫）
   - Phase 3：多 Skill 協作（`/meeting @zhangsan @lisi`）
   - Phase 4：多模態（照片、語音克隆、影片）

---

## 7. 建議的後續行動

### 技術債嚴重程度矩陣

```mermaid
quadrantChart
    title 技術債嚴重程度 vs 修復難度
    x-axis 修復難度低 --> 修復難度高
    y-axis 嚴重程度低 --> 嚴重程度高
    quadrant-1 立即處理
    quadrant-2 規劃處理
    quadrant-3 觀察追蹤
    quadrant-4 長期改善
    Secrets 明文儲存: [0.35, 0.75]
    PRD.md 文件過時 D-01~D-05: [0.15, 0.45]
    ruff 非阻塞 lint: [0.12, 0.35]
    yt-dlp 依賴未說明: [0.20, 0.30]
    transcribe_audio 後端未說明: [0.18, 0.28]
    feishu_browser 無測試覆蓋: [0.65, 0.55]
    dingtalk_auto 無測試覆蓋: [0.68, 0.50]
    MCP Node.js 依賴未說明: [0.22, 0.32]
    budget-unfriendly 流程缺教學: [0.40, 0.42]
```

### 7.1 維護者可優先處理的改善

**立即（低難度高價值）：**

1. **更新 `docs/PRD.md`**：同步 D-01 至 D-05 落差項目，避免新貢獻者依據過時文件開發
2. **補充 `INSTALL.md` Celebrity 章節**：說明 `yt-dlp`、`faster-whisper` 的安裝指令
3. **將 ruff 改為阻塞式**：移除 `.github/workflows/ci.yml` 中的 `|| true`，或配置 `ruff.toml` 白名單以控制阻塞範圍

**短期（中難度中價值）：**

4. **Secrets 安全性改善**：考慮整合 OS keychain（`keyring` Python 套件）儲存 API token，替換明文 JSON
5. **採集器測試覆蓋**：為 `feishu_auto_collector.py`、`slack_auto_collector.py`、`email_parser.py` 加入 Mock-based 單元測試

**長期（高難度）：**

6. **`feishu_browser.py` 穩定性**：Playwright selector 可能因飛書 DOM 更新失效，需定期維護策略

### 7.2 新貢獻者入手建議

1. **從測試開始理解架構**：
   - `tests/test_skill_writer.py` → 了解 Skill artifact 生命週期
   - `tests/test_cli_lifecycle.py` → 了解完整 CLI 流程

2. **閱讀順序建議**：
   ```
   SKILL.md（了解控制平面）
     ↓
   tools/skill_presets.py（了解 Character 系統）
     ↓
   tools/skill_schema.py（了解元數據正規化）
     ↓
   tools/skill_writer.py（了解執行層）
   ```

3. **容易入手的貢獻：**
   - 補充 `docs/PRD.md` 的過時描述（純文件修改，無需執行環境）
   - 為 `tools/email_parser.py` 新增測試（邏輯相對獨立，無需外部 API）
   - 在 `INSTALL.md` 補充 `yt-dlp` 安裝說明

4. **避免直接動核心程式碼**：`skill_writer.py` 的 `merge_markdown_patch()` 和 `apply_correction()` 是核心算法，修改前需確保測試完整覆蓋

---

*本文件由 dot-skill 偵察流程自動產出，最後更新：2026-04-27*
