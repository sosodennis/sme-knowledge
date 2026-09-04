# SME Knowledge Base

## 工程師入口

實作時請先閱讀唯一規範文件：

- [SME 知識庫工程實作規格](<./SME-知識庫工程實作規格.md>)：current normative baseline，直接作為開發、測試和 MR 驗收依據。
- [English engineering specification](<./SME-Knowledge-Base-Engineering-Spec.md>)：英文 companion；中文規範優先，文字有差異時以中文為準。

以下文件只作研究證據和背景，不是第二套需求：

- [調研校準與實現路線圖](<./SME-知識庫調研校準與實現路線圖.md>)：來源核查、方案比較和取捨記錄。
- [English research calibration and roadmap](<./SME-Knowledge-Base-Research-Calibration-and-Roadmap.md>)：英文研究 companion，不是需求文件。
- [已驗證來源](<./docs/verified-sources.md>)：官方文件和固定 commit 的研究證據。
- [Skills/Agents 深度調研](<./docs/skills-agents-research-20260903.md>)：Skills、Custom Agents、handoff、guardrails 與 pressure tests 的研究證據。
- [Legacy / noise inventory](<./docs/legacy-inventory.md>)：已淘汰方向與清理決策。
- [WikiSkill 論文第三方審計](<./docs/wikiskill-paper-audit-20260904.md>)：論文主張、可重現性、第三方實作與採納邊界。

目前核心運行基線是：Git canonical files、Python、SQLite FTS5、Graph-lite relations、read-only local MCP、VS Code Copilot 的 `sme-router` → bounded workers proposal workflow、GitLab single-MR governance 和 post-merge deterministic rebuild。演進層採用 redacted experience、pattern、skill-history 和 iteration records，但仍是 proposal-first、人工批准；raw trace 不進 runtime。Embedding、LanceDB vector、GraphRAG、Ragas、LLM API pipeline 都不是目前實作依賴。

流程圖和圖片已在架構契約中預留：Mermaid 以文字和結構化 process 欄位保存，raster/vector 圖以 asset sidecar、hash、provenance 和 alt/summary 保存；目前只計劃檢索這些文字 metadata，不能宣稱已具備像素語義搜尋。repository 目前仍未落地 `knowledge/`、`schemas/`、compiler 或 MCP 實體，需按工程規格 Phase 1–3 實作後才可運行。

可重用程式碼以 `knowledge/snippets/<snippet-id>.md` 保存：frontmatter 記錄語言、版本、依賴、用途、來源、授權、測試和安全狀態，正文保留一個 exact fenced code block；多檔案範例拆成多個 snippet 並以 relation 連結。初版以 SQLite FTS5 做 identifier/import/API 的 lexical 檢索；`get_snippet` 只返回受控純文字和 metadata，compiler、CI、MCP 和 Agent 都不自動執行 snippet。這是已定義的工程契約，實體目錄、schema 和實作仍按 Phase 1–3 落地。
