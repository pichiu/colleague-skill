# Web 搜尋發現報告

## 搜尋說明

本專案為 colleague-skill / dot-skill，已在 README.md 中提供完整的外部資源連結。根據現有文件資訊整理如下，無需額外 web search（sub-agent 無法使用 web search）。

## 外部資源索引

| 資源 | 連結 | 說明 |
|------|------|------|
| GitHub Repo | https://github.com/titanwings/colleague-skill | 主倉庫，截至 2026-04-19 已達 15k Stars |
| 社群 Gallery | https://titanwings.github.io/colleague-skill-site/ | 社群提交的 Skill 展示，100+ Skills |
| AgentSkills 標準 | https://agentskills.io | 本專案遵循的 Agent Skill 開放標準 |
| Discord 社群 | https://discord.gg/NVX66RxWZv | 即時交流頻道 |
| Karpathy Skill 案例 | https://github.com/alchaincyf/karpathy-skill | 社群貢獻的 celebrity Skill 範例 |
| 技術報告 PDF | colleague_skill.pdf（repo 根目錄） | Colleague.Skill 論文：Work Skill + Persona 雙層架構 |

## 關鍵發現摘要

### 專案背景
- 原名 `colleague-skill`，於 2026-04-13 宣布升級為 `dot-skill`
- 由 @titanwings 個人開發，由 Shanghai AI Lab + AI Safety Center 提供算力支持
- 從「只能蒸餾同事」擴展到「蒸餾任何人」

### 技術架構發現
- 遵循 AgentSkills 開放標準：整個 repo 是一個 skill 目錄，SKILL.md 是入口
- 支援 4 個 Agent 宿主：Claude Code / Hermes / OpenClaw / Codex
- Work Skill + Persona 雙層架構有對應技術論文支撐

### 社群採用狀況
- 15k GitHub Stars（截至 2026-04-19）
- 社群 Gallery 100+ Skills
- 7 種語言的 README 翻譯（DE/EN/ES/JA/KO/PT/RU）
- WeChat 社群（已開到第 9 群）

### Roadmap 重點
- Phase 2：dot-skill 通用化，`/create-skill` 萬用入口
- Phase 3：多 Skill 協作，`/meeting @zhangsan @lisi @wangwu`
- Phase 4：多模態（照片、語音克隆、影片）

### 已知限制（來自 README/INSTALL）
- Slack 免費版限制 90 天訊息記錄
- 釘釘 API 不支援歷史訊息，需瀏覽器採集
- Word/Excel 需手動轉換
- DingTalk 私聊採集限制
