# SME 知識庫（VS Code GitHub Copilot）調研校準、架構與實現路線圖

> **文件定位：研究／證據附錄，不是工程實作規範。**
>
> 工程師實作、測試和 MR 驗收請以 [SME 知識庫工程實作規格](<./SME-知識庫工程實作規格.md>) 為唯一依據。本文件保留調研過程、替代方案、未來能力和被拒絕的方向，避免研究證據遺失；其中的 `future`、`optional`、`不建議基線`、`移除` 和 `Open Questions` 不得當成目前需求。

**研究日期：** 2026-09-04
**目標場景：** Android 專案；在 VS Code 內使用 GitHub Copilot Chat/Agent（不自行整合模型 API），維護一個可審查、低複雜度、可持續演進的個人或內部 SME 知識庫。
**對照原稿：** 原稿是本研究的輸入材料，未隨本 repository 發佈；本文件本身和下方來源檔是可移植的研究記錄。

英文 research companion：[SME Knowledge Base Research Calibration, Architecture, and Roadmap](<./SME-Knowledge-Base-Research-Calibration-and-Roadmap.md>)。如中英文有差異，以本中文研究記錄及中文工程規範為準。

## 1. 執行摘要

這個項目可行，但原稿把產品已具備的能力、需要自行開發的能力、以及仍屬於研究選項的技術混在了一起。研究結論是：第一版交付固定為 `core-lexical`（SQLite FTS5 + Graph-lite relations + read-only MCP），同時保留 LanceDB/embedding adapter seam；只有未來 provider 通過資源、品質和治理驗證後，才建立獨立 semantic/hybrid build。這是「單一真相、可降級」的分階段架構，而不是目前同時維護兩套索引：

```text
Git 知識源（Markdown/YAML/Mermaid/圖片/程式碼片段）
        │
        ├── 確定性驗證與編譯（source → stable chunk IR）
        │       ├── SQLite 控制面（current）：metadata、FTS5、relations、provenance、狀態
        │       └── LanceDB 語義面（future adapter）：provider ready 時才保存 embedding、ANN/相似度候選
        │
        ├── VS Code Agent Skills（按需載入檢索與維護流程）
        ├── VS Code Custom Agents（Router、角色、工具權限、handoff）
        └── 本機只讀 MCP stdio Server（current lexical search；future semantic/hybrid + provenance gateway）
                │
                └── GitHub Copilot 在 VS Code 內取得證據後完成推理與回答
```

目前環境不能使用 embedding，因此第一個 Android 試點以 `core-lexical` 交付：SQLite FTS5、結構化關係、provenance 和 MCP 先完整運作。LanceDB 保留 adapter/schema 和未來遷移位置，但暫不進入 query path；只安裝 LanceDB 而沒有 embedding model，不會自動產生語義檢索能力。`EmbeddingProvider` 抽象和 `DisabledEmbeddingProvider` 空實現先固定接口，日後取得模型批准後再切換到 `hybrid`。圖資料庫、Ragas、Leiden 仍不應作為前置條件。

網上幾種 GraphRAG/Property Graph 方案顯示，值得吸收的是「顯式 typed edges、由搜尋命中點向外受控遍歷、路徑和證據可解釋」，而不是必然引入常駐圖資料庫。Microsoft GraphRAG 的標準索引會用 LLM 抽取 entity/relation/claim、Leiden 建社群、生成 community reports 並建立 embedding；這與目前沒有 LLM API key 的 GitLab pipeline 不相容。故本方案新增 **Graph-lite** 方向：Git 中的 entity/relation 仍是唯一真相，確定性編譯到 SQLite `relations`，以 recursive CTE 做最多兩層的 bounded traversal，再由 MCP 返回路徑和 provenance。日後若有 embedding，向量只負責找 seed，仍沿用同一個圖遍歷和權威過濾契約。

最重要的邊界是：

1. **Git 中的知識源是 SSOT；SQLite/LanceDB/報告都是可刪除、可重建的衍生產物。**
2. **AI 可以提議新增或修改實體、規則、關係，不能未審批直接改寫 canonical knowledge。**
3. **「Copilot 非 API」代表 GitLab CI 不能在背後自動呼叫 Copilot 模型。** 當前不做批量 LLM 抽取、LLM evaluator、API 型 reranker 或任何需要 API key 的 pipeline；Copilot 只在 VS Code 互動式協助起草 proposal，所有 CI/build 檢查保持 deterministic。
4. **MCP 標準能傳遞 image content，但不保證每一個 Copilot host/model 都以視覺輸入處理它。** 所有圖片必須同時有可檢索的文字摘要、OCR/標註（若適用）和來源鏈接。
5. **Kuzu 目前不適合作為新項目的預設依賴。** 其 GitHub repository 已於 2025-10-10 封存；原稿的「可直接採用 Kuzu」需要撤回。

## 2. 研究方法與判定標籤

本報告優先使用 GitHub、VS Code、Model Context Protocol、Agent Skills、Android Developers、LanceDB、SQLite 和 Mermaid 的官方文檔或官方 repository；Ragas 僅作為「為何暫不納入 LLM evaluator pipeline」的背景核查，不是當前依賴。每個主張按以下標籤處理：

| 標籤 | 含義 |
|---|---|
| **已核實** | 官方資料直接描述，或本機可重現的基礎能力。 |
| **有條件** | 協議/套件支持，但受 Copilot host、模型、版本、環境、授權或外部服務限制。 |
| **需自行實作** | 不是 VS Code、MCP 或資料庫自動提供的功能。 |
| **不建議基線** | 可做研究或遷移，但增加不必要複雜度、風險或維護負擔。 |

本機探測結果（2026-09-03）：

```text
Python 3.12.12       Node v26.5.0       npm 11.17.0
uv 0.11.28 (optional) Git 2.39.5         ripgrep 15.2.0
SQLite FTS5: available                  Docker: /opt/homebrew/bin/docker
mmdc: not found                         VS Code CLI (code): not found
```

## 3. 對原稿方向的校準表

| 原稿方向/主張 | 判定 | 校準後方向 | 原因與證據 |
|---|---|---|---|
| Git + Markdown/YAML/Mermaid 作為知識源 | **已核實/推薦** | 保留；把所有衍生索引放入 `.cache/` 並在 Git ignore | 可 diff、可 review、可回滾；Mermaid 官方定位就是以文字/程式碼定義圖表。 |
| `.github/skills/sme-advisor/SKILL.md` | **已核實但格式需修正** | 改為兩個有清晰觸發邊界的 core Skills：`sme-kb-retrieve` 和 `sme-kb-maintenance`；兩者都使用合法 Agent Skills frontmatter | VS Code 和 Agent Skills specification 都要求 `SKILL.md`、`name`、`description` 等格式；把檢索與寫入流程分開可避免 Skill 權限和上下文膨脹。 |
| 以 `SKILL.md` 取代所有知識 | **不完整** | Skill 只放工作流程和行為約束；領域事實仍放 `knowledge/` | Skill 會按相關性和漸進式揭示載入，適合「怎樣做」，不適合承載整個 Android 文檔庫。 |
| 自訂 SME Agent | **已核實** | 使用 `.github/agents/*.agent.md`；以低權限 Router 作日常入口，按角色限制 worker 工具，使用 handoff/MR 讓人批准階段轉移 | VS Code 支持工具清單、subagents 和 handoffs；handoff 是可控的引導流程，不是無人監管的自動工作流。 |
| 本機 FastMCP/stdio MCP | **有條件/推薦本機只讀版** | 優先使用官方 `mcp` Python SDK；server 只暴露查詢工具和資源 | VS Code 支持本機 stdio MCP，但本機 server 可執行任意程式，首次啟動要信任；需限制路徑、輸入、輸出和工具權限。 |
| MCP 返回 `ImageContentBlock` 就能讓 Copilot 看圖 | **有條件，原稿過度承諾** | 圖片提供原件 + alt text/摘要 + 結構化節點；視覺結果只能在已驗證的 host/model 組合使用 | MCP spec 定義 image content（base64 + MIME），但協議能力不等於特定 Copilot host/model 的視覺推理保證。 |
| LanceDB 作為第一天的向量/多模態資料庫 | **有條件/預留，不作當前 runtime 基線** | 目前以 `core-lexical` 啟動；保留 LanceDB adapter/schema，只有在 embedding provider `ready` 且 benchmark 通過後才啟用 vector path | LanceDB OSS 確實是 embedded 本地庫，也能同表存向量、metadata 和 binary；但需要另行選擇 embedding model，存圖本身不會產生跨模態檢索。 |
| SQLite 作為關係/KV 緩存 | **已核實/推薦** | 直接作為第一版 metadata、provenance、FTS5、顯式關係存儲 | SQLite 是 in-process、serverless、zero-configuration、單文件、支持 SQL/索引/交易。 |
| Kuzu 作為嵌入式圖資料庫 | **不建議基線** | 新項目不採用；保留「未來評估替代品」研究項 | Kuzu GitHub repository 已封存/只讀。封存項目不應成為新內部平台的核心運行依賴。 |
| Ragas merge gate，Faithfulness < 0.95 直接阻斷 | **目前移除** | 只做 deterministic checks + 人工 golden set；Ragas/LLM-as-judge 不進當前 GitLab pipeline，日後若有獨立服務授權再作隔離研究 | Ragas 通常需要 evaluator LLM；分數受資料集、模型和 prompt 影響，不是事實真相。 |
| Leiden 自動把文件移到新領域目錄 | **需自行實作/不建議自動化** | 只作離線 modularization audit；輸出建議報告或草稿 MR，不直接搬移 canonical files | `leidenalg` 是圖分群函式庫，依賴 igraph，輸入是明確圖；它不懂業務邊界，也不能證明目錄重構正確。 |
| Mermaid 正則提取完整 AST/三元組 | **需自行實作** | CI 先做 render/syntax check；結構化節點只抽取明確支持的 diagram 類型，必要時採用 parser 或人工關係欄位 | Mermaid 是 renderer/語法工具；任意圖類型的完整語義抽取不是 Mermaid 自動承諾的能力。 |
| 模型自動反熵巡檢並提出修復 MR | **有條件/延後** | 先做 stale-review report；任何修改只能產生 draft proposal，必須 SME review | 需要外部模型/API、來源可信度、成本控制、版權/資料外洩政策和反錯誤機制；Copilot Chat 本身不是可在 CI 直接調度的 API。 |
| 新模型變強後，知識庫自動幾何級成長 | **不應如此表述** | 把「可重編譯、可重評估、可重檢索」設為成長能力；模型升級帶來的是條件式收益 | 新模型可改善抽取、重排或回答，但只有在 provenance、schema、評估集和編譯版本可追溯時，才可證明質量提升。 |
| 增量編譯可使大型庫保持固定 10–50ms | **方向正確，數字無證據** | 使用 content hash 和變更文件集合；以本機 benchmark 決定性能門檻，不預先承諾延遲 | 真實延遲受磁碟、索引、文件量、模型和 MCP 啟動方式影響；原稿數字應刪除。 |

## 4. 最終建議架構

### 4.1 分層責任

| 層 | 內容 | 是否 canonical | 主要責任 |
|---|---|---:|---|
| 知識源層 | Markdown、YAML frontmatter、Mermaid、圖片、fenced code snippet、來源副本/鏈接 | 是 | 人類可讀、可 diff、可審批、可回滾 |
| 確定性編譯層 | frontmatter 驗證、ID/alias 正規化、heading 切塊、hash、FTS5、關係表 | 否 | 把源文件轉成可查詢快照；不自行發明事實 |
| 索引控制面 | SQLite metadata、FTS5、relations、provenance、build manifest | 否 | 權威過濾、狀態和證據定位；所有查詢結果必須回到此處核對 |
| 語義索引面 | 本地 embedding、LanceDB、可選 reranking | 否 | 只保存由 source/chunk hash 對應的衍生向量；失效時退回 FTS5 |
| MCP gateway | `search`（lexical/semantic/hybrid）、`get_rule`、`get_document`、`get_snippet`、`list_related`、`get_asset` | 否 | 統一協調兩個索引；不把任一資料庫直接暴露給 Copilot；只讀優先 |
| Copilot customization | `copilot-instructions.md`、path instructions、Skills、Custom Agents | 否 | 定義何時查、怎樣引用、怎樣提出變更 |
| 治理與 CI | schema/lint、死鏈、Mermaid render、golden cases、CODEOWNERS | 否 | 阻止壞資料合入，記錄評估和版本 |

### 4.2 推薦 repository 結構

```text
android-sme-kb/
├── .github/
│   ├── copilot-instructions.md       # 全庫最小、常駐規則
│   ├── instructions/
│   │   └── knowledge.instructions.md  # applyTo: knowledge/**/*.md
│   ├── agents/
│   │   ├── sme-router.agent.md        # 日常入口；只委派 allow-listed workers
│   │   ├── android-advisor.agent.md
│   │   ├── kb-curator.agent.md
│   │   └── kb-reviewer.agent.md
│   ├── skills/
│   │   ├── sme-kb-retrieve/
│   │   │   ├── SKILL.md
│   │   │   └── references/retrieval-contract.md
│   │   └── sme-kb-maintenance/
│   │       ├── SKILL.md
│   │       ├── references/lifecycle.md
│   │       └── scripts/propose_change.py
├── knowledge/
│   ├── sources/                       # 權威來源清單、版本、license、抓取日期
│   ├── entities/                      # 穩定名詞與 canonical ID
│   ├── rules/                         # 原子規則；每條可獨立 review
│   ├── decisions/                     # 公司決策/ADR；與官方建議分開
│   ├── processes/                     # Mermaid + 步驟文字 + 例外
│   ├── assets/                        # PNG/JPEG/SVG；大型檔案可 Git LFS
│   ├── snippets/                      # 可獨立引用的 fenced code + metadata
│   ├── proposals/                     # AI/工程師草稿；不直接進 active 索引
│   ├── conflicts/                     # open/resolved conflict records
│   └── taxonomy.yml                   # predicate、domain、authority 的受控詞彙
├── schemas/
│   ├── common.schema.json             # id/status/authority/scope/source
│   ├── source.schema.json
│   ├── entity.schema.json
│   ├── rule.schema.json
│   ├── relation.schema.json
│   ├── process.schema.json
│   ├── decision.schema.json
│   ├── asset.schema.json
│   ├── snippet.schema.json
│   ├── proposal.schema.json
│   ├── conflict.schema.json
│   ├── taxonomy.schema.json
│   ├── chunk-ir.schema.json
│   └── build-manifest.schema.json
├── src/sme_kb/                        # 一個本機 Python package，不拆微服務
│   ├── contracts.py                    # typed input/output 與 ID/hash 契約
│   ├── config.py                       # retrieval/model/policy 設定載入
│   ├── compiler.py                     # parse → normalize → stage build
│   ├── stores/
│   │   ├── sqlite_store.py             # 控制面、FTS5、relations、provenance
│   │   └── lancedb_store.py            # 向量/多模態衍生索引
│   ├── retrieval.py                    # lexical + semantic + hybrid fusion
│   ├── embedding.py                    # 本地 embedding provider 介面/版本檢查
│   ├── provenance.py                   # source/chunk locator 組裝
│   ├── security.py                     # 路徑、MIME、limit、timeout 驗證
│   └── mcp_server.py                   # 官方 mcp SDK；stdio；只讀 gateway
├── scripts/
│   ├── validate_kb.py                 # 純確定性驗證
│   ├── build_indexes.py                # 一次建立 SQLite + LanceDB snapshot
│   ├── check_index_consistency.py     # chunk/hash/model/manifest 對帳
│   ├── inspect_relations.py            # 顯式關係/死鏈/孤立節點報告
│   ├── benchmark_retrieval.py         # lexical/hybrid A/B 和 p95
│   └── report_maintenance.py           # stale/duplicate/conflict 報告
├── configs/
│   ├── retrieval.yml                   # top-k、fusion weights、filters
│   ├── models.yml                      # embedding model、dimension、checksum
│   └── policy.yml                      # authority、MIME、size、timeout、資料分類
├── tests/
│   ├── golden_cases.yml                # 人工核准的問題、期望來源、禁答條件
│   ├── fixtures/                       # 小型 canonical source fixture
│   ├── test_contracts.py
│   ├── test_retrieval.py
│   ├── test_index_consistency.py
│   └── test_mcp_security.py
├── .cache/                             # gitignored；可完全刪除重建
│   ├── builds/<build-id>/
│   │   ├── manifest.json
│   │   ├── sqlite/index.sqlite         # 控制面快照
│   │   └── lancedb/                    # 語義面快照
│   └── current.json                    # atomic pointer；不直接改 current build
├── pyproject.toml                     # package metadata/dependency declarations
├── requirements.txt                   # pip baseline lock（若選 Poetry/uv/Conda，使用其等價 lock）
├── requirements-dev.txt               # pip baseline dev/CI lock（可由團隊工具替代）
└── .gitlab-ci.yml                     # deterministic schema/build/security pipeline
```

這個目錄保留「一個本機 package、一個 MCP process、兩個 embedded store」的低複雜度；不把它拆成服務網絡。`knowledge/` 中的顯式 relations 是唯一關係真相，SQLite 保存控制面編譯結果，LanceDB 只保存帶有 `chunk_id + content_sha256 + embedding_model` 的衍生向量。不要同時維護 `relations.yml`、SQLite graph、Kùzu graph 和向量 graph 四份相同真相；任何圖分群結果是報告，不是 canonical 分類。

### 4.3 Copilot customization 分工

**`copilot-instructions.md`：** 只保留約一至兩頁的全局規則，例如「回答 Android 專案問題先查 `knowledge/`；所有硬性規則附 ID 和來源；找不到證據要明說；不要把 proposal 當成已批准規則」。不要把完整 Android 文檔貼進常駐提示。

**Skills：** 僅放可重複的工作流程，並按風險和觸發情境分成兩個 core Skills：`sme-kb-retrieve` 負責查詢、引用、bounded relation traversal 和降級回報；`sme-kb-maintenance` 負責 source intake、proposal、lifecycle、validate、dry-run 和 handoff。兩者都可以附 `scripts/`、`references/` 和 `assets/`，但 `SKILL.md` 只放入口說明、限制和步驟，詳細 schema 放 references。不要為每個 entity 或 Android 子領域各建一個 Skill。

**Custom Agent：** 放角色、工具權限和輸出格式：

- `sme-router.agent.md`：唯一建議的日常入口；分類 query/maintenance/review/mixed，按 allow-list 委派，檢查 worker 回覆和 handoff，遇到問題停下或升級。
- `android-advisor.agent.md`：只讀檢索和回答，不能修改文件。
- `kb-curator.agent.md`：讀取來源並建立 proposal，允許編輯 proposal 分支，不得直接改 `main`。
- `kb-reviewer.agent.md`：檢查規則衝突、來源和 Android 版本適用範圍，優先只讀工具。

所有檢索型呼叫都經同一個 read-only MCP gateway；curator 另外使用受限的 workspace edit 和 deterministic scripts 產生 proposal，reviewer 只讀 proposal/報告。gateway 內部再協調 SQLite 和可選的 LanceDB；任何 Agent 都不應直接知道 LanceDB table 名稱、SQL schema 或 embedding 維度，這些屬於 infrastructure detail。當前 advisor 的 `search_knowledge` 預設使用 lexical；只有 `get_build_info` 報告 provider `ready` 且 hybrid 通過 benchmark 後，才可切換預設。若 semantic index 不可用，應自動標示 `retrieval_mode: lexical_fallback`，而不是把失敗隱藏掉。

Router 以 VS Code subagent 機制做 bounded delegation：使用者通常只需選 Router；低風險 query 可以自動委派 advisor，maintenance 可以依次委派 curator → reviewer，結果和 handoff 在同一個可見 session 返回。高風險寫入、purge 和 MR approval 仍由人手完成，不使用 `send: true` 讓流程越過人手 gate。Router 的固定 transitions、最大 4 hops、每 stage 一次重試和 context/hash 檢查詳見[Skills/Agents 深度研究](<./docs/skills-agents-research-20260903.md>)。

**Router 的責任邊界：** Router 應承擔委派健康檢查（錯誤分類、未授權 target、worker 超時、空/矛盾輸出、handoff 遺失、循環和 context loss），並以 `clarify`/`blocked` 停止或升級；它不承擔 domain semantic truth，也不修補 canonical、替 reviewer 批准或自行解決 conflict。worker 負責角色內工作，Python validator/CI 負責機械正確性，SME/CODEOWNER 負責 semantic approval。即使未來模型路由能力提升，這個責任分層不變。

### 4.4 Router 的實作狀態機

Router 不需要新的常駐服務或模型 API；它是一個 VS Code Custom Agent，利用
Copilot 既有的 `agent`/subagent 能力。工程師應把以下狀態和限制直接寫入
Agent 指示，並在 worker 輸出和 CI 中重複檢查：

```text
RECEIVE
  → CLASSIFY(query|maintenance|review|mixed|clarify|blocked)
  → PLAN(allow-listed target, hop=0, retry=0)
  → INVOKE(stateless worker with complete context envelope)
  → CHECK(status, IDs, source commit, hashes, scope, validation)
       ├─ query success → RETURN cited answer
       ├─ maintenance success → INVOKE reviewer → RETURN MR-ready status
       ├─ changes-required (once) → INVOKE same curator with report
       ├─ ambiguous/contract failure/timeout → STOP with reason code
       └─ human gate → RETURN; SME/CODEOWNER reviews one MR
```

固定規則：`agents` 只列 `android-advisor`、`kb-curator`、`kb-reviewer`；
每個 request 最多 4 hops、每 stage 一次 retry、不可遞迴 subagent；每次
委派帶 `route_id`、`hop`、`visited_agents`、source commit、target IDs、
`base_hash`、scope 和 expected output fields。`confidence` 只能用於觀察和
golden-case 評估，不能觸發 merge、purge 或自動批准。若 Router 自身不可靠，
使用者可從 Customizations editor 暫時顯示 worker 作人工 fallback，但這不
改變 worker 的最小權限和同一 MR 治理。

## 5. 知識內容究竟要提取什麼

不要把每個名詞都做成 entity，也不要把每段摘要都升格為 rule。判斷標準是：該資料是否會改變檢索、決策、代碼審查或流程推導。

| 類型 | Android 例子 | 最小內容 | 是否必須 AI |
|---|---|---|---:|
| Entity/concept | `StateFlow`、`ViewModel`、`repeatOnLifecycle`、`Repository`、`HiltViewModel` | canonical ID、名稱、別名、定義、適用版本、來源 | 否；AI 可提議 |
| Relation | `UI_COLLECTS_WITH`、`VIEWMODEL_EXPOSES`、`REPOSITORY_OWNS` | source ID、predicate、target ID、scope、證據 | 否；可由文件引用或 AI 提議 |
| Rule | UI 要更新畫面時，Flow 應在 `repeatOnLifecycle` 內收集 | 條件、要求/建議、例外、反例、來源、有效性/檢視日期 | 否；AI 只起草 |
| Decision/ADR | 公司規定 Data layer 處理網絡與資料庫 dispatcher | 背景、決策、替代方案、影響、owner、review date | 否；可由會議紀錄提取初稿 |
| Process | Activity STARTED/STOPPED 時的收集流程 | Mermaid、步驟、分支、失敗處理、關聯 rules | Mermaid 文字可人工寫；AI 可起草 |
| Evidence | Android Developers 頁面、公司 guideline、版本化 API reference | URL/檔案、retrieved date、locator、license、hash | 否 |
| Asset | 架構圖、截圖、掃描流程圖 | 原圖、alt text、摘要、OCR/節點（若可信）、來源 | 否；VLM/OCR 只是可選輔助 |
| Snippet/code | Kotlin/Java/Python/Bash 範例、設定或 patch | exact fenced body、language/type、版本與依賴、用途/限制、tested/security status、source/hash | 否；Copilot 可互動式起草，compiler/CI 不執行 |

此表中的「AI 可提議」只表示工程師在 VS Code 內使用 GitHub Copilot 互動式起草 proposal；不表示 GitLab CI 可以調用 LLM API，也不表示任何 proposal 可以跳過 SME/CODEOWNER 審批。

### 5.1 程式碼片段的保存與檢索邊界

可重用、需要獨立引用或需要版本/安全審查的程式碼，使用 `knowledge/snippets/<snippet-id>.md`。YAML frontmatter 保存 `language`、`language_version`、`snippet_type`（`example`、`template`、`command`、`config`、`patch`、`pseudocode`）、`purpose`、`when_to_use`、`not_for`、`dependencies`、`framework_versions`、`tested_on`、`source`、`license`、`provenance`、`security_review` 和 `review_by`；正文只放一個 fenced code block，保存 exact code。短的說明性片段可以留在 rule/process 正文，但可複製範例應升格為 snippet。

Baseline 是一個 snippet 對應一個可獨立引用的單檔 code body；多檔案範例拆成多個 snippet，再以 `knowledge/relations/` 連結，不另造 bundle 格式。

Compiler 只解析 frontmatter、計算 `content_sha256`，並把 metadata 和 code body 產生為 `modality: code` 的 SQLite FTS5 chunks。它不執行 shell、SQL、build script、Kotlin/Python 或依賴下載；CI 只做 schema、path/license、secret/credential 和危險命令檢查。語言 parser/linter/compiler（例如 `tree-sitter` 或 Kotlin compiler）只有在 toolchain 獲批准時才作 optional check，失敗不能把 snippet 標成 tested。

目前以 lexical search 查找 identifier、import、API 名稱和註解，再以 `get_snippet` 返回 exact code、metadata、source、hash、compatibility、tested/security status 和 build ID；若正文來自 derived chunk，必須先與 Git canonical file/build manifest 對帳，hash 不一致就回報 index inconsistency。沒有 embedding 也不影響代碼的精確查詢；未來 semantic code search 只能透過既有 `EmbeddingProvider` seam 和獨立 LanceDB candidate build 評估。Retrieved code 是不受信任資料，Agent 不得自動執行或把 example/template 升格為 project Rule；任何執行要求都要先由人手確認並在受控 sandbox 進行。

Snippet 沿用既有 `create/update/retire/restore/purge` 和單一 MR。只有語義、版本和授權都可保留時才提出 `merge`，否則保留兩個 ID；snippet 與 Entity/Rule 的關係放在 `knowledge/relations/`，例如 `illustrates`、`implements`、`supersedes`。

### 5.2 Android 規則範例

Android 官方文檔明確指出：UI 需要更新時，不應直接用 `launch` 或 `launchIn` 收集 Flow，應用 `repeatOnLifecycle`；其文檔也說明 `repeatOnLifecycle` 可用於 `lifecycle-runtime-ktx:2.4.0` 及更高版本。這可被建模為一條可引用規則，而不是把整篇文檔壓成一個向量：

```markdown
---
id: rule.android.flow.ui-collection
kind: rule
title: UI 收集 Flow 時遵循生命週期
status: active
authority: official
scope:
  platform: android
  applies_to: ["UI", "StateFlow", "SharedFlow"]
validity:
  introduced_by: "androidx.lifecycle:lifecycle-runtime-ktx:2.4.0"
  review_by: 2027-01-01
source:
  - url: https://developer.android.com/kotlin/flow/stateflow-and-sharedflow
    locator: "Warning: Never collect a flow from the UI directly from launch or launchIn"
entities: [concept.android.stateflow, concept.android.repeat-on-lifecycle]
relations:
  - type: recommends
    target: concept.android.repeat-on-lifecycle
---

## 條件

當 Flow 的收集結果會更新 UI，且 UI 具有 STARTED/STOPPED 生命週期時，適用本規則。

## 要求

使用 `repeatOnLifecycle(Lifecycle.State.STARTED)` 包裹收集區塊；不要直接以 `launch`/`launchIn` 讓不可見 UI 持續處理事件。

## 例外與邊界

本條不是所有背景工作或非 UI consumer 的通用規則。需要由具體 scope 和 lifecycle 證明適用性。

## 反例

`lifecycleScope.launch { viewModel.uiState.collect { render(it) } }`
```

### 5.3 Entity、relation、rule 的關係

在上例中：

- `StateFlow`、`repeatOnLifecycle`、`UI` 是 entity。
- 「UI 收集 Flow 時使用 repeatOnLifecycle」是帶條件的 rule。
- `UI --COLLECTS_WITH--> repeatOnLifecycle` 是可查詢 relation。
- 官方頁面段落是 evidence。

這四者互相鏈接，但不應把同一句話複製到四個文件中。複製會造成更新時的矛盾；用 ID 和 provenance 連接即可。

## 6. AI 生成、修改和衝突治理

### 6.1 允許與禁止的操作

| 操作 | AI 可否直接做 | 建議流程 |
|---|---:|---|
| 生成來源摘要、候選 entity、alias、relation | 可提議 | 產生 `proposals/` 或分支文件，附原文片段和來源定位 |
| 建立新的 rule 草稿 | 可提議 | 必須包括條件、例外、反例和 authority；由 SME review |
| 修改已批准 rule | 不可直接合入 | 生成 patch/MR，保留舊版本、理由和 supersedes |
| 發現兩條 rule conflict | 可以報告 | 建立 conflict record；停止「自動折衷」，交由 owner 決定 |
| 自動把兩個 entity 合併 | 不可直接合入 | exact alias 可自動標記候選；語義相似只能建立候選對 |
| 把文件搬到新領域目錄 | 不可自動合入 | modularization report → 人工批准 → 腳本執行可回滾 patch |
| 更新 canonical source | 不可無人審批 | 僅允許 proposal branch，main 受 branch protection/CODEOWNERS 保護 |
| 更新 `.cache/` 索引 | 可以 | 由確定性編譯腳本從 Git source 重建，不提交或只提交可再生 manifest |

### 6.2 Entity resolution（相似內容歸類）

第一版採用可解釋的分層流程：

1. **標準化：** Unicode/大小寫/空白/標點處理；保存原始 mention。
2. **確定性 alias：** `androidx.work.WorkManager`、`WorkManager`、公司約定縮寫可在 alias registry 中直接映射。
3. **詞法候選：** SQLite FTS5/normalized exact lookup 找 Top-N 候選。
4. **可選 embedding 候選：** 只有在 benchmark 證明詞法不足時，使用本地 embedding 或 LanceDB 找候選；相似度閾值只代表「待審核候選」，不是自動合併證據。
5. **仲裁：** Copilot 的 curator agent 讀取候選、來源上下文和 entity 定義，輸出 `merge | keep_separate | subtype | unknown` 及理由。
6. **審批：** SME 只批准 proposal；批准後才更新 alias/relations，並保留 decision log。

例子：`CoroutineWorker` 不應因為名字相似就合併為 `WorkManager`。合理結果可能是獨立 entity，並以 `USED_BY` 或 `SUBTYPE_OF` 連至 `WorkManager`；這是領域語義判斷，不是 cosine threshold 能單獨決定的事。

### 6.3 Conflict record

衝突不是錯誤地「刪掉其中一條」，而是知識的一部分：

```yaml
id: conflict.android.dispatcher-001
kind: conflict
status: open
claims:
  - source_id: decision.company.dispatcher
    statement: ViewModel 不直接指定 IO dispatcher
  - source_id: article.community.dispatcher
    statement: ViewModel 可使用 withContext(Dispatchers.IO)
reason: authority_or_scope_difference
resolution:
  owner: android-platform-team
  due: 2026-10-01
  decision: null
```

回答 Agent 遇到 `status: open` 必須說明衝突和適用範圍，不能產生看似確定的折衷答案。

### 6.4 WikiSkill 演進操作與既有 lifecycle 的關係

吸收 WikiSkill 後，不需要重寫 Entity/Rule 的 `create/update/retire/merge/split/restore/purge/resolve_conflict` 語義。新增的是獨立的 evolution record 操作，目的在保存經驗和演進 Skill，而不是把 experience 當成 domain truth：

| 演進操作 | 允許產物 | 明確禁止 |
|---|---|---|
| `record_experience` | redacted、hash-addressed experience summary | raw transcript、secret、hidden chain-of-thought 或 active fact |
| `propose_pattern` | 帶 evidence/uncertainty 的 pattern candidate | 直接 active 或覆寫 Entity/Rule |
| `propose_skill_change` | Skill diff、iteration/history manifest | 改 routing permission、直接 activate 或 bypass MR |
| `validate_evolution` | schema、hash、regression/golden report | 把 `validated` 當成人手批准 |
| `activate_evolution` | merge/build 後的 active projection | 由 Copilot、Router 或 CI 在 MR 前執行 |
| `reject_evolution` / `supersede_evolution` | 可追溯的歷史結果和 replacement link | 物理刪除理由或隱藏失敗記錄 |

Curator 可以起草，Reviewer 可以檢查；只有同一個 GitLab MR 的 SME/CODEOWNER approval、CI pass 和 Maintainer merge 才能使 evolution record 變成 `active`。因此 `activate_evolution` 不是 Agent/MCP write tool，而是 post-merge 狀態投影。若 evolution record 要改動 Entity、Rule 或 Relation，仍必須另在同一 MR 中包含該 domain lifecycle proposal、evidence、base hash 和 deterministic relation mapping。

## 7. 多模態設計

### 7.1 優先順序

1. 流程、時序、狀態機、架構拓撲：優先 Mermaid source。
2. 截圖、掃描文件、歷史圖：保留原圖，旁邊建立同名 metadata/alt text。
3. 任何圖片都要有文字描述和 provenance；不要假定模型一定可讀原圖。
4. 當 MCP host 支持 image content 時，再返回圖片；不支持時返回摘要、OCR（若經核實）、Mermaid/結構化節點和檔案路徑。

### 7.2 圖片 sidecar 範例

```yaml
id: asset.android.navigation-v2
kind: asset
path: knowledge/assets/navigation-v2.png
mime_type: image/png
alt_text: "單一 Activity 容器承載 Compose destinations；UI state 由 ViewModel 提供。"
summary: "導航容器、ViewModel、Repository 和 DataSource 的依賴方向圖。"
visual_entities: [concept.android.activity, concept.android.viewmodel, concept.android.repository]
source:
  url: https://developer.android.com/topic/architecture
  retrieved_at: 2026-09-03
review:
  status: verified_summary
  reviewer: android-platform-team
```

LanceDB 的 multimodal column 可以存 image bytes，但「能存」和「能跨模態檢索」是兩件事。後者需要選定並驗證 image/text embedding 模型、向量維度、模型 license、CPU/GPU 成本、索引版本，以及 Copilot 端如何消費返回內容。

### 7.3 現況與實作邊界

架構層面已預留流程圖和圖片的 canonical contract；但截至本研究日期，repository 仍只有規格和研究文件，尚未實作 `knowledge/assets/`、`knowledge/processes/`、`schemas/asset.schema.json`、`schemas/process.schema.json`、compiler 或 MCP server。因此「系統可以處理」目前應理解為「設計可支援，完成 Phase 1–3 後可運行」，不是今天 checkout 後已能直接 ingest 圖片。

流程圖應保存 Mermaid source 或人工轉錄的結構化 `steps`、`branches`、`failure_handling` 和 typed relations；PNG/JPEG/WebP/SVG 應以 `asset.yml` sidecar 保存 path、MIME、content hash、alt text、summary、provenance、license 和 review status。CI 可檢查 Mermaid syntax/render、檔案大小、MIME magic bytes、SVG script/external-reference、path traversal、敏感資料和死鏈。原圖是展示/證據，文字 metadata 才是目前可檢索 fallback。

PDF、音訊、影片等其他二進位檔也只能按 asset sidecar 和個別 MIME allow-list 接受；沒有已核實的文字轉錄時，只作受控 archival evidence，不會自動變成可查詢的知識。

無 embedding 時，compiler 只把 Mermaid/process text、asset title/alt/summary/已核實 OCR，以及 snippet metadata/code body 編成 SQLite FTS5 chunks；不讀像素、不做 image similarity，也不需要 Pillow/OCR/model。`get_asset` 可返回受控 resource link，`get_snippet` 返回 exact code 和 metadata；Copilot 是否真正理解 image content 取決於實際 MCP host/model。未來若要跨模態或語義 code search，才以 `EmbeddingProvider` + 獨立 LanceDB candidate build 評估相應模型；這不會改變 canonical asset/snippet 或 human/MR gate。

## 8. MCP 技術實現規格

### 8.1 最小只讀工具面

| 工具 | 輸入 | 輸出 | 安全規則 |
|---|---|---|---|
| `search_knowledge` | query、`mode: lexical\|semantic\|hybrid`、kind、domain、status、modality、limit | snippets、IDs、fusion score、各檢索器分數、source paths、`retrieval_mode` | limit/top-k 上限；只查索引；不接受任意 SQL；semantic/hybrid 失效時明示 fallback |
| `get_document` | canonical ID 或受控相對路徑 | 正文、frontmatter、provenance | 禁止 `..` 路徑穿越；限制檔案大小 |
| `get_rule` | rule ID | 條件、要求、例外、來源、review 狀態 | 不存在/過期/衝突要明確返回狀態 |
| `list_related` | entity/rule ID、depth（最多 2） | 顯式關係邊和節點摘要 | 不執行任意 graph query |
| `get_asset` | asset ID | alt text、摘要、resource link；可選 image content | MIME allow-list；限制 bytes；無圖像 fallback |
| `get_snippet` | snippet ID | exact fenced code、language/type、dependencies、compatibility、tested/security status、source、hash、build ID | 純文字返回；不能由 MCP 執行；限制輸出大小 |
| `get_build_info` | 無 | source commit、schema/compiler version、SQLite build、LanceDB build、embedding model/dimension、index timestamp、degraded flags | 幫助回答附版本 provenance；兩個索引 build 不一致時禁止宣稱 hybrid 完整可用 |

工具輸出建議同時返回人類可讀 text 和結構化 JSON；MCP spec 支持 output schema，client 可驗證結果。所有工具保持 read-only；提議修改走本機腳本/MR，不暴露 `write_file` 給普通顧問 Agent。

### 8.2 VS Code 配置概念

工作區 MCP 配置放在 `.vscode/mcp.json`，可納入 Git 共享命令和參數，但不要硬編碼 API key。配置示意：

```json
{
  "servers": {
    "sme-kb": {
      "type": "stdio",
      "command": "python",
      "args": ["-m", "sme_kb.mcp_server"],
      "cwd": "${workspaceFolder}",
      "env": {"SME_KB_ROOT": "${workspaceFolder}"}
    }
  }
}
```

實際欄位、server name 和 VS Code 版本要在你的安裝環境中以 IntelliSense/官方配置 reference 驗證。VS Code 第一次啟動本機 server 會要求信任；本機 server 可執行任意程式，應審查依賴、鎖定 working directory，並在 macOS/Linux 評估 `sandboxEnabled`。

`command: "python"` 代表使用當前 PATH 中已安裝依賴的 Python。若 VS Code 沒有繼承已啟用的虛擬環境，應改成該環境的絕對 Python 路徑（例如 `.venv/bin/python`；Windows 使用 `.venv\\Scripts\\python.exe`），或由團隊的環境啟動腳本提供一致入口。這只是執行環境選擇，不把 MCP server 綁定到特定依賴管理器。

### 8.3 Hybrid search 的責任邊界

MCP server 內部的查詢順序建議如下：

1. 解析 query 和 filters；`status`、`authority`、domain、版本範圍以 SQLite 控制面為準。
2. 平行執行 SQLite FTS5 lexical search 和 LanceDB semantic search。semantic search 使用本地 embedding provider；Copilot 不需要、也不應直接持有模型 API key。
3. 以 `chunk_id` 合併候選。可用 Reciprocal Rank Fusion（RRF）或經 benchmark 決定的加權融合；RRF 可表示為 `score(d)=Σ_i w_i/(k+rank_i(d))`，其中 `k` 和各檢索器權重放在 `configs/retrieval.yml`，不是寫死在 prompt，也不是跨語料通用的真理閾值。
4. 回到 SQLite 讀取 canonical chunk、rule/entity 狀態、locator、authority 和 source；若 LanceDB metadata 與 SQLite 不一致，以 SQLite 為準並記錄 `index_stale`。
5. 返回短 snippets、evidence IDs、source paths、`retrieval_mode` 和 score breakdown；只有 `get_document`/`get_rule`/`get_snippet` 才按需展開全文或 exact code。

LanceDB 不應直接成為 Agent 的第二個資料源。它回答「哪些 chunk 可能相關」，SQLite 和 Git 才回答「這個 chunk 是否 active、適用什麼 scope、證據在哪裡」。若 LanceDB build 失敗，gateway 返回 lexical-only 結果和 degraded flag；若 SQLite 控制面不可用，則整個 MCP 查詢失敗，不能使用沒有 provenance 的向量結果。LanceDB 本身也支援不依賴 embedding 的 BM25 Full-Text Search，但這會和 SQLite FTS5 重複；沒有特定 benchmark 或 LanceDB FTS 功能需求時，不要同時查兩套 lexical index。第一版也不要再同時加入 `sqlite-vec`/`sqlite-vss` 類 SQLite vector extension；那會形成第三套向量索引和另一套 ABI/重建路徑，只有在未來 benchmark 證明需要單文件部署時才另行評估。

## 9. 確定性編譯與增量更新

### 9.1 編譯流程

```text
掃描 knowledge/
  → 讀 YAML frontmatter，Pydantic/JSON Schema 驗證
  → 以 Git-relative path + id 建立穩定主鍵
  → heading/段落切塊，保留 line/heading locator
  → 寫入 stable chunk IR（document/chunk/entity/relation/source）
  → 建立 SQLite 控制面：metadata、FTS5、relations、provenance、狀態
  → 若 provider status=ready，建立 LanceDB 語義面；否則使用 DisabledEmbeddingProvider 並跳過 vector build
  → 對帳 chunk_id/content_sha256/model/schema，再發布 immutable build manifest
```

每個 chunk 記錄：`document_id`、`chunk_id`、`source_path`、`heading`、`text`、`content_sha256`、`authority`、`status`、`review_by`。修改時只重建 hash 改變的文件；若 schema、切塊算法、embedding model 或 LanceDB index strategy 改變，增加 `compiler_version`/`index_version` 並允許全量重建。SQLite 和 LanceDB 都從同一份 stable chunk IR 產生，不能各自重新切塊。

### 9.2 SQLite schema（最小版）

```sql
CREATE TABLE documents (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL UNIQUE,
  kind TEXT NOT NULL,
  title TEXT NOT NULL,
  status TEXT NOT NULL,
  content_sha256 TEXT NOT NULL,
  source_commit TEXT NOT NULL
);

CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  document_id TEXT NOT NULL REFERENCES documents(id),
  heading TEXT,
  body TEXT NOT NULL,
  locator TEXT NOT NULL,
  FOREIGN KEY (document_id) REFERENCES documents(id)
);

CREATE VIRTUAL TABLE chunks_fts USING fts5(
  chunk_id UNINDEXED, title, heading, body, aliases,
  tokenize = 'unicode61'
);

CREATE TABLE relations (
  source_id TEXT NOT NULL,
  predicate TEXT NOT NULL,
  target_id TEXT NOT NULL,
  evidence_chunk_id TEXT,
  PRIMARY KEY (source_id, predicate, target_id)
);

CREATE TABLE index_builds (
  build_id TEXT PRIMARY KEY,
  source_commit TEXT NOT NULL,
  compiler_version TEXT NOT NULL,
  sqlite_schema_version TEXT NOT NULL,
  lancedb_schema_version TEXT,
  embedding_model TEXT,
  embedding_dimension INTEGER,
  status TEXT NOT NULL
);

-- Optional semantic-profile table; core-lexical does not create or populate it.
CREATE TABLE chunk_vectors (
  chunk_id TEXT NOT NULL REFERENCES chunks(id),
  content_sha256 TEXT NOT NULL,
  embedding_model TEXT NOT NULL,
  dimension INTEGER NOT NULL,
  lancedb_table TEXT NOT NULL,
  PRIMARY KEY (chunk_id, embedding_model)
);
```

SQLite 控制面和 LanceDB 語義面都放在 `.cache/builds/<build-id>/`，不是 Git SSOT。編譯器先寫 staging build，再執行一致性檢查，最後只替換 `.cache/current.json` 指針；MCP 永遠讀一個完整 build，不在 current build 原地更新。若同一倉庫被多個程序同時編譯，使用 temporary path + atomic rename，避免讀到半成品。

### 9.3 雙索引發布與一致性不變量

每個已發布 build 至少要滿足：

- SQLite `documents/chunks` 的每個 active chunk 有穩定 `chunk_id` 和 `content_sha256`。
- 只有 semantic/hybrid build 才要求 LanceDB 向量；每一筆向量都能在 SQLite `chunks` 找到相同 `chunk_id` 和 hash，找不到的向量視為 orphan，不得返回給 Agent。
- vector metadata 的 `embedding_model`、dimension、modality、compiler/index version 與 manifest 一致。
- status、authority、scope 和 review date 只由 SQLite 控制面判定；LanceDB metadata 只作 prefilter 優化，查詢後仍要回 SQLite 核對。
- `current.json` 至少指向同一個 `build_id` 的 SQLite snapshot；只有 hybrid build 才同時要求對應的 LanceDB 目錄。core-lexical 可以明確記錄 `semantic_available=false`，不因沒有 LanceDB vector snapshot 而視為失敗。

LanceDB 的最小 row 契約是：`chunk_id`、`vector`、`content_sha256`、`build_id`、`embedding_model`、`dimension`、`modality`。不要把 `authority` 或 `status` 複製成可獨立決策的欄位；若為了 prefilter 暫存，必須由一致性檢查確認它仍與 SQLite 相同。

`check_index_consistency.py` 應輸出 `missing_vectors`、`orphan_vectors`、`hash_mismatch`、`model_mismatch`、`manifest_mismatch` 五類結果；`missing_vectors` 只在 manifest 宣告 semantic/hybrid 時作為覆蓋率問題。core-lexical 不建立 vector table，也不能把它誤報為完整 hybrid 覆蓋率。

### 9.4 增量編譯策略

以 `content_sha256`、`schema_version`、`chunking_version` 和 `embedding_model` 組成重建 key：

| 變更 | SQLite | LanceDB | 發布策略 |
|---|---|---|---|
| 正文/metadata hash 改變 | 重建該文件的 chunks、relations、FTS rows | 只重算受影響 chunk 向量 | staging build 對帳後 atomic promote |
| 只改 prompt/Agent | 不重建 | 不重建 | 只跑回答 regression |
| 改 chunking/schema | 全量重建受影響表 | 全量重建 vectors | 新 build 與舊 build A/B |
| 改 embedding model/dimension | 不改 canonical SQLite | 新建獨立 LanceDB table/build | candidate 通過 benchmark 後切 stable |
| LanceDB 暫時不可用 | 可用 | 標 `semantic_unavailable` | 發布 lexical-only build 或保留上一個完整 build |

### 9.5 Schema 設計狀態與實作邊界

目前的答案是：**MVP 的資料契約已在本路線圖中完成欄位級設計，但 repository 尚未實際建立和驗證全部 `schemas/*.schema.json` 文件。** 目前的 SQL 是索引資料庫 schema，Markdown/YAML 範例是資料契約草案；兩者不能被誤稱為已完成的正式 JSON Schema。正式實作時，所有 canonical YAML/frontmatter 都要經同一套 `$ref`、版本和 fixture 驗證。

| Schema | 對應資料 | MVP 必須欄位/約束 | 當前狀態 |
|---|---|---|---|
| `common.schema.json` | 所有 canonical 文件共用欄位 | `id`、`kind`、`title`、`status`、`authority`、`scope`、`source`、`review_by`；`id` 全庫唯一 | 欄位契約已設計；待落地 JSON Schema |
| `source.schema.json` | `knowledge/sources/` | URL 或受控本地 path、title、locator、`retrieved_at`、license、content hash/使用限制 | 欄位契約已設計；待落地 |
| `entity.schema.json` | `knowledge/entities/` | canonical name、aliases、definition、scope、source、version/status | 欄位契約已設計；待落地 |
| `rule.schema.json` | `knowledge/rules/` | condition、requirement、exceptions、counterexamples、entities、evidence、authority | 有 Markdown 範例；待落地 JSON Schema |
| `relation.schema.json` | entity/rule 顯式關係 | source、predicate、target、scope、evidence；禁止未知 ID 和非法自循環 | 欄位契約已設計；待落地 |
| `process.schema.json` | `knowledge/processes/` | steps、branches、failure handling、diagram/source locator | 已列入目錄；待落地 |
| `decision.schema.json` | `knowledge/decisions/` | context、decision、alternatives、impact、owner、review date、evidence | 已列入目錄；待落地 |
| `asset.schema.json` | `knowledge/assets/` | path、MIME、alt text、summary、provenance、可選 OCR/visual entities | 有 sidecar 範例；待落地 |
| `snippet.schema.json` | `knowledge/snippets/` | language、snippet type、purpose/when_to_use/not_for、dependencies/framework versions/tested_on、source/license/provenance、security review；正文單一 fenced block | 欄位契約已設計；待落地 |
| `proposal.schema.json` | `knowledge/proposals/` | target、operation、proposed patch、evidence、uncertainty、author、review status | 有流程契約；待落地 |
| `conflict.schema.json` | `knowledge/conflicts/` | claim IDs、差異類型、owner、due date、resolution、open/resolved status | 有 YAML 範例；待落地 |
| `taxonomy.schema.json` | `knowledge/taxonomy.yml` | 受控 `kind/status/authority/predicate/domain` 值；禁止未登記枚舉 | 已定義方向；待落地 |
| `chunk-ir.schema.json` | compiler 內部 stable chunk IR | document/chunk ID、source path、heading、text、locator、`content_sha256`、modality、compiler version | 建置契約已設計；待落地 |
| `build-manifest.schema.json` | `.cache/builds/<build-id>/manifest.json` | source commit、schema/compiler/chunking versions、SQLite snapshot、semantic status/provider、artifact hashes | 欄位契約已設計；待落地 |
| `evolution-experience.schema.json` | `knowledge/evolution/experiences/` | trace hash、task/outcome、agent/skill/model metadata、redaction status、evidence refs | WikiSkill 擴充契約；Phase 5 待落地 |
| `evolution-pattern.schema.json` | `knowledge/evolution/patterns/` | pattern statement、evidence、uncertainty、validation refs、status | WikiSkill 擴充契約；Phase 5 待落地 |
| `evolution-iteration.schema.json` | `knowledge/evolution/iterations/` | input trace hashes、pattern/Skill diff hashes、validation report、source commit、review status | WikiSkill 擴充契約；Phase 5 待落地 |
| `skill-history.schema.json` | `knowledge/evolution/skill-history/` | Skill/proposal IDs、base/candidate hashes、outcome、validation/rejection/supersedes | WikiSkill 擴充契約；Phase 5 待落地 |

正式 schema 的實作順序應是 `common → source/entity/rule/relation → proposal/conflict → process/decision/asset/snippet/taxonomy → chunk-ir/build-manifest → evolution-experience/evolution-pattern/evolution-iteration/skill-history`。每完成一組，就加入 valid/invalid fixtures 和 GitLab CI validator；在此之前，路線圖中的 schema 只能視為 implementation-ready draft，不能宣稱「所有 schema 已完成」。

## 10. 質量、來源和評估

### 10.1 CI 必做門禁（不需模型 API）

- YAML/frontmatter schema 合法、ID 唯一、`name`/路徑一致。
- `status`、`authority`、`source`、`review_by` 欄位完整。
- relation source/target 存在；禁止自循環（除非 schema 明確允許）。
- Markdown 內部鏈接和圖片鏈接無死鏈。
- Mermaid 文件能用 pinned `mmdc` render；若環境未安裝，CI 要明確 fail 或標為 skipped，不可假稱通過。
- source license/URL/retrieved date 存在；不把未授權全文直接鏡像進內部 repo。
- 生成 `.cache` 後，`get_build_info` 可追溯 source commit 和 compiler version。

### 10.2 Golden set

從 20 個高價值 Android 問題開始，每題保存：

```yaml
id: case.flow-collection-001
question: "Activity 中如何收集 ViewModel 的 StateFlow？"
required_sources:
  - rule.android.flow.ui-collection
must_mention: [repeatOnLifecycle]
must_not_claim: ["所有 Flow 都必須在 UI 用 repeatOnLifecycle"]
acceptable_answer_notes: "要區分 UI 收集與背景 consumer。"
```

當前 MVP 只用來源命中率、required source 命中、禁止錯誤 assertion、人工抽樣評審、recall@k（以人工標註為準）和 p50/p95 MCP latency 做基線；先測 `lexical`，未來有 provider `ready` 才增加 `hybrid` A/B。也記錄 SQLite/LanceDB 磁碟大小、（未來）embedding 建置時間和峰值 RAM。這些指標不需要 LLM API；GitLab CI 不安裝或執行 Ragas、LLM-as-judge 或任何外部 evaluator。Copilot 可以在 VS Code 互動式協助解讀失敗案例，但結果由人工確認，不作自動 MR gate。

### 10.3 反熵策略

第一階段只做「需要人工確認的報告」：

- `review_by` 到期文件清單。
- 官方來源 URL 失效/版本變更提示。
- 新提交與現有 canonical entity 的候選重複。
- conflicting rules 和沒有 owner 的 open conflict。
- 未被任何 golden case 或最近查詢命中的高風險規則。

自動修復 MR 和批量 LLM 抽取目前從 roadmap/pipeline 移除。知識新增、摘要、entity 候選和 conflict 草稿只可由工程師或 VS Code Copilot 互動式產生 proposal，再由 SME/CODEOWNER 審批；未來若取得獨立模型服務授權，才另開研究項，不改變當前 GitLab CI 基線。

### 10.4 GitLab CI 基線（不含 LLM API）

GitLab pipeline 只負責可重跑、可審查的 deterministic 工作：schema/frontmatter 驗證、Python 單元測試、`core-lexical` SQLite build、索引一致性和可選的 Mermaid render。它不安裝或呼叫 Copilot、OpenAI/Anthropic API、Ragas evaluator、批量抽取器或 hosted reranker，也不要求任何 LLM secret。

最小示意（實際 job image、lock 安裝方式和 artifact policy 按公司 GitLab runner 調整）：

```yaml
stages: [validate, test, build]

default:
  image: python:3.12-slim
  before_script:
    - python -m venv .venv
    - .venv/bin/python -m pip install --upgrade pip
    - .venv/bin/python -m pip install --require-hashes -r requirements-dev.txt

validate:
  stage: validate
  script:
    - .venv/bin/python scripts/validate_kb.py

test:
  stage: test
  script:
    - .venv/bin/python -m pytest

build_lexical:
  stage: build
  script:
    - .venv/bin/python scripts/build_indexes.py --profile core-lexical
    - .venv/bin/python scripts/check_index_consistency.py --profile core-lexical
  artifacts:
    expire_in: 1 week
    paths:
      - .cache/builds/
      - .cache/current.json
```

如果團隊選擇 Poetry、uv、Conda 或其他依賴管理器，只替換 `before_script` 的環境安裝和 lock restore；`validate_kb.py`、`build_indexes.py`、`check_index_consistency.py` 的 Python 入口和 `core-lexical` 行為保持不變。`.cache/` artifact 可以只作測試輸出，不應變成 canonical source 或必須提交到 Git。

### 10.5 當前 Pipeline 取捨

| 原先構想 | 當前決定 | 替代做法 |
|---|---|---|
| GitLab job 批量呼叫 LLM 抽取 entity/rule | 移除 | 工程師在 VS Code 使用 Copilot 起草 proposal；人工 review 後提交 Git |
| Ragas/LLM-as-judge quality gate | 移除 | deterministic source-hit、schema/link checks、golden cases 和人工評審 |
| Hosted embedding 或 API reranker | 移除 | `DisabledEmbeddingProvider`、SQLite FTS5；日後只接受已批准的本地 provider |
| VLM/OCR API 自動解析圖片 | 移除 API pipeline | 人工 alt text/summary/OCR sidecar；本地 OCR 只有在另行批准後才加入 |
| Copilot 自動寫入 canonical knowledge | 禁止 | Copilot 只能產生 proposal；SME/CODEOWNER 通過 MR 後才合入 |

因此，當前 GitLab CI 的唯一智能來源不是模型，而是 Git 中已審批的 canonical knowledge；Copilot 的作用限於使用者主動操作時的互動式整理和草稿生成。

## 11. 依賴庫與環境依賴清單

### 11.1 第一版推薦依賴

| 依賴 | 用途 | 必需性 | Runtime/語言 | 環境檢查/注意 | 授權/服務考量 |
|---|---|---:|---|---|---|
| Git | SSOT、分支、MR、版本追蹤 | 必需 | 系統工具 | `git --version`；啟用 branch protection/CODEOWNERS | Git client GPLv2；GitLab/企業託管政策另計 |
| Python 3.10+ | validator/compiler/MCP server | 必需 | Python | 官方 MCP Python SDK 目前要求 Python 3.10+；本機 3.12.12 合適 | Python PSF；版本由企業 runtime policy 管理 |
| Python 虛擬環境 + 依賴鎖定 | 可重現 Python 環境 | 必需 | Python/系統工具 | 基線使用 `python -m venv` + `python -m pip`；可按公司政策改用 Poetry、uv、Conda 等，但必須提交一份可重建的 lock/requirements 文件 | 依賴管理器本身不是架構耦合點；registry/快取政策和 transitive license 需確認 |
| `mcp`（官方 Python SDK） | MCP tools/resources/prompts、stdio | 必需（current baseline） | Python | SDK repository 的 v2 是目前 stable line；v2 對 v1 有 breaking changes，鎖定 major/minor 並測試 | 官方 SDK MIT；MCP server 仍需企業 trust/資料政策 |
| `pydantic` | frontmatter/提議資料的 typed validation | 推薦 | Python | 以 schema 驗證 canonical input；不讓模型輸出直接寫庫 | MIT；以 lock 中版本及 transitive license 審查 |
| `PyYAML` | YAML frontmatter/sidecar 解析 | 推薦 | Python | 需處理 YAML 解析安全；只用 safe loader | MIT；禁止對不可信輸入使用 unsafe loader |
| `jsonschema` | CI schema 驗證 proposal/資料 | 推薦 | Python | schema 文件本身納入版本控制 | MIT；schema 和 validator 版本要可追溯 |
| Python `sqlite3` + SQLite FTS5 | metadata、provenance、全文索引、顯式關係 | 必需 | Python/系統庫 | 本機已確認 FTS5 available；CI 要再探測 | SQLite public domain；FTS5 編譯選項需在 runner 再驗證 |
| `lancedb` | 未來語義/多模態衍生索引，與 SQLite 以 `chunk_id` 對帳 | 當前 core profile 可不啟動；future hybrid 或受控 FTS POC 才需要 | Python | 先建立小型 adapter/目錄 POC；確認本地目錄、PyArrow/平台 wheel、讀寫鎖和磁碟空間 | OSS Apache-2.0；只保存 derived vectors，模型/資料 license 另審 |
| 本地 embedding provider（推薦 `sentence-transformers`；`fastembed` 只在所選模型出現在其 supported list 時使用） | 為文本 chunk/query 生成向量；支援未來 LanceDB semantic search | 當前不可用；`hybrid` profile 才必需 | Python/本地 CPU 或 GPU | 先只實作 provider interface + disabled adapter；日後記錄 model ID、dimension、revision/checksum、下載來源和 CPU/RAM；不要同時引入兩個 provider | 套件與模型權重 license 按 pinned release 審查；不需要外部 API key |
| `pytest` | compiler/MCP/schema 單元測試 | 開發必需 | Python | 對 path traversal、空索引、過期規則、錯誤輸入寫測試 | MIT；只作 development/CI dependency |
| `ruff` | Python lint/format | 推薦 | Python | CI 固定版本 | MIT/Apache-2.0；固定版本避免格式漂移 |
| Node.js + `@mermaid-js/mermaid-cli` | Mermaid render/syntax CI | 只在使用 Mermaid CI 時必需 | Node | 本機有 Node 26，但 `mmdc` 未安裝；固定 package/container 版本 | Node MIT、Mermaid CLI MIT；npm registry 或審批 container 需可用 |

### 11.2 按需加入的依賴（當前不安裝）

以下項目全部是未來研究或明確的可選項，不會進入目前的 GitLab CI 或 runtime lock；尤其不包含任何需要 LLM API key 的 pipeline。

| 依賴/服務 | 何時加入 | 代價與風險 | 授權/服務考量 |
|---|---|---|---|
| reranker（可先不加入） | hybrid 召回後仍有大量近似候選，且 benchmark 證明需要二階重排時 | 先用 RRF/簡單加權；新增 reranker 要測量 p95、RAM 和錯誤案例 | 模型 license、推理資源和資料政策需另審；不可改 canonical truth |
| `fastembed` | 選定模型確實在其 supported list，且需要較輕量 ONNX/CPU runtime 時 | 不假設它支援所有 Hugging Face embedding；以官方 supported-models 頁面和 pinned release 核對；BGE-M3/E5-base/small 不應未驗證就直接指定 | fastembed + ONNX Runtime；首次下載仍需 cache；模型/維度仍要對帳 |
| `Pillow` | 圖片尺寸/MIME/縮圖/完整性檢查 | 只做檔案處理，不會自動理解圖片 | HPND；僅處理資產，不提供 VLM 能力 |
| OCR/VLM（如 Tesseract 或獲批准的本地視覺模型） | 圖片中的文字是不可替代證據，且人工標註成本過高 | OCR/VLM 需人工抽樣驗證；不要把低置信 OCR 直接升格為 rule | Tesseract/模型 license 和本地權重來源必須個別審查 |
| `leidenalg` + `python-igraph` | 離線審計 entity/relation graph 的社群和模組邊界 | C++/igraph 依賴、內存和版本管理；只輸出建議，不直接自動搬文件 | 以實際 pinned release license 做法務審查；不放入 runtime |
| Git LFS | 圖片/掃描 PDF 大到不適合一般 Git clone | 需確認 GitHub/企業托管配額、備份、CI checkout 行為 | Git LFS GPLv2；託管容量與流量可能收費 |
| `pre-commit` | 本地提交前統一執行 YAML/schema/Markdown checks | 增加開發者初始化步驟；CI 仍需獨立執行同樣檢查 | MIT；hook 來源需 pin/審查 |
| GitLab CI/CD | MR lint、render、deterministic build、consistency、報告 | 當前 pipeline 不呼叫 Copilot、LLM API、Ragas 或外部 evaluator；runner 的 Python/Node 版本需 pin | GitLab runner/registry 政策、第三方 image 和 job token 權限需審查 |

### 11.3 明確不列為基線

| 項目 | 原因 |
|---|---|
| Kuzu | repository 已 archived/read-only；不應作新核心依賴。 |
| Neo4j/Milvus/常駐向量叢集 | 與個人/內部低複雜度和本機 Copilot 目標不匹配；除非有多使用者高吞吐需求。 |
| LightRAG 作為必需 runtime | 它是完整 RAG 取捨，不是 entity schema、compiler 或 MCP 的必要依賴；當前先驗證自有 SQLite lexical retrieval，未來才評估 hybrid。 |
| `Instructor` 作為必要抽取層 | 當前不使用；結構化輸入由 Pydantic/JSON Schema 驗證，proposal 由 Copilot 互動式起草後人工審批。 |
| LLM API 型 pipeline | 當前沒有 API key；暫不安裝或執行 Ragas/LLM-as-judge、批量 LLM 抽取、hosted reranker、VLM API；由 Copilot 互動式產生 proposal，人工審批後才入庫。 |
| 固定 0.88/0.95/10–50ms 閾值 | 原稿沒有針對你的語料、模型和硬件的 benchmark 證據。 |

## 12. Roadmap

| 階段 | 交付內容 | Exit criteria | 主要風險/回滾 |
|---|---|---|---|
| 0. 環境與政策（1–2 日） | 確認 VS Code/Copilot plan、MCP 是否允許、Python/Node、SQLite、LanceDB package 和公司資料/版權政策；記錄 embedding 暫不可用 | 能在本機執行 Python、SQLite FTS5、`DisabledEmbeddingProvider` contract test；確認是否可使用 `.github/skills`、`.github/agents`、`.vscode/mcp.json`；選定 `core-lexical` | embedding 不可用不阻塞知識建模；保留 provider seam，日後再做模型批准和 hybrid benchmark |
| 1. Schema 試點（3–5 日） | 建立 `sources/entities/rules/processes/assets/snippets`；輸入 10–20 份 Android 官方/公司文件；人工核准 20–40 條 rule/entity 和至少一個 snippet | 每條 rule/snippet 有 ID、scope、authority、來源定位、review date；snippet 單一 fenced body、依賴/測試/安全 metadata 完整；schema/死鏈/ID CI 通過 | 發現來源版權問題時只保留鏈接和短摘錄，刪除未授權副本；禁止把未核實代碼標成 tested |
| 2. Copilot UX（2–4 日） | `copilot-instructions.md`、兩個 core Skills、`sme-router`，以及 advisor/curator/reviewer 三個 bounded workers | 在 VS Code 能發現 Skills 和 Agents；使用者通常只選 Router；Router 正確委派 query/maintenance/review，對 ambiguous intent、worker failure、hash/context 遺失停止；advisor 回答附 rule ID/source；curator 只產生 proposal；reviewer 只輸出 checklist | Skill/Router 未載入時先用 `/skills`、Agent Customizations editor 和 Agent Debug Logs 檢查；不把全庫塞入 prompt；若某項 host metadata 不支援，仍由 MCP/腳本/分支保護 enforce；不要開啟 recursive subagents |
| 3. Compiler、provider seam 與 MCP（4–7 日） | `validate_kb.py`、stable chunk IR、SQLite 控制面、LanceDB adapter interface、`DisabledEmbeddingProvider`、官方 `mcp` Python SDK 只讀 gateway；編譯 fenced snippets | `core-lexical` 的 `search_knowledge` 可用；`modality: code` 和 `get_snippet` 返回 exact code/metadata；`check_index_consistency.py` 通過；`get_build_info` 顯示 `provider: disabled`；未來 `semantic/hybrid` 保留為 capability-gated mode；無任意 SQL/path traversal 或 code execution | 不載入模型、不建立空向量表、不執行 snippet；MCP trust 或 host 不可用時保留 CLI search |
| 4. 多模態與圖表（3–5 日） | Mermaid render CI；圖片 sidecar、alt text、資產 provenance；MCP image + fallback POC | Mermaid 語法錯誤會被 CI 發現；有圖/無圖 client 都能取得等價文字上下文 | 視覺結果不可靠時關閉 image content，只返回摘要和結構化節點 |
| 5. 評估與增量編譯（1–2 週） | 20–50 golden cases、hash manifest、lexical regression、人工回歸、索引一致性和資源 benchmark；未來 provider ready 才增加 hybrid A/B | 能比較 source-hit、recall@k、禁答、人工品質、p95 latency、RAM/磁碟；只重建 changed files；可發布 lexical-only build | 質量退化時切回上一個 build；不使用 LLM judge 或單一模型分數自動合入 |
| 6. 語義增強（未來按需，1–2 週 POC） | 取得模型政策批准後更換本地 embedding、加入本地 reranker、圖片向量或 query routing；和 lexical baseline A/B | 新組件能改善召回/duplicate candidate quality，且版本、license、磁碟/RAM/重建成本可接受；不依賴 hosted LLM API | POC 無收益就停用；任何新向量表都可刪除並由 source 重建 |
| 7. 模組審計（按需） | 顯式關係圖匯出、`leidenalg`/igraph 離線報告 | SME 確認社群結果能解釋，且報告能指出孤立/過度耦合節點 | 不自動移動文件；以 draft MR 和可回滾腳本處理 |
| 8. 受控成長（持續） | 來源更新報告、conflict queue、CODEOWNERS、模型/編譯器版本矩陣 | 每次知識變更可追溯；新模型只有通過 golden/regression 才成為預設 | 外部模型/API 失效時仍可用 deterministic compiler 和現有索引 |

## 13. 具體日常工作流

下面的流程是把「知識維護」限制在可審查的 Git 變更上。Copilot 可以協助搜尋、整理和產生 proposal；`validate_kb.py`、`build_indexes.py` 和 MR checks 負責確定性檢查；SME 或指定 CODEOWNER 負責最後批准。任何流程都可以在沒有 MCP 的情況下用本地腳本和 workspace 文件完成。

### 13.1 新增一條 Android 規則

| 步驟 | 操作者/工具 | 產物與驗證 |
|---|---|---|
| 1. 建立 intake | 工程師或 curator agent | 記錄問題、適用 Android/Jetpack 版本、候選來源和預期讀者；先標 `draft` |
| 2. 登記來源 | 人工 + `knowledge/sources/` | 保存 URL、頁面標題、版本/發布日期（若有）、`retrieved_at`、license/使用限制和段落 locator；不要未授權鏡像整頁 |
| 3. 寫原子 rule | 人工或 agent 起草 | 每條 rule 只回答一個可審查決策；包含 `condition`、`requirement`、例外、反例、scope、authority、`review_by`、entity IDs 和 evidence |
| 4. 產生 proposal | `proposals/` 或 MR 分支 | proposal 必須列出新增/修改/刪除、原文短摘錄、推理和不確定點；不得直接覆蓋 `active` rule |
| 5. 執行確定性檢查 | `python scripts/validate_kb.py`（使用已選定的虛擬環境） | YAML/frontmatter、ID、日期、來源、關係、內部鏈接、路徑和禁止字段通過；失敗即修正，不靠模型自行判定通過 |
| 6. SME review | CODEOWNERS/MR | 審查 authority、scope、例外、版本適用性、是否和現有 rule 衝突；批准後將狀態改為 `active` |
| 7. 編譯與冒煙查詢 | `python scripts/build_indexes.py`、MCP `search_knowledge`/`get_rule` | 以暫存 `core-lexical` SQLite build（或已通過 gate 的 future hybrid build）編譯後 atomic pointer；確認 rule ID、來源 locator、關係和 retrieval mode 可被查到 |
| 8. 記錄版本 | Git commit/MR | build manifest 記錄 source commit、schema/compiler/index 版本；必要時加入 golden case，讓後續模型升級有回歸基線 |

推薦的規則生命週期是 `draft → proposed → active → superseded → retired`。`superseded` 要指向取代它的 rule；`retired` 仍保留退役理由和歷史來源，避免回答 Agent 把舊規則當成現行規則。

最小 proposal 例子：

```yaml
id: proposal.rule.android.flow.ui-collection-20260903
kind: rule_proposal
operation: add
target_id: rule.android.flow.ui-collection
source_excerpt: " repeatOnLifecycle ... collect ... UI ..."
proposed_scope:
  platform: android
  applies_to: [UI, StateFlow, SharedFlow]
uncertainties:
  - "背景 service consumer 不應套用 UI lifecycle 限制"
requested_reviewers: [android-platform-team]
```

### 13.2 發現重複 Entity

重複發現是候選生成，不是合併命令。每日或每個 MR 可以執行下列報告：

```text
canonical ID/alias normalization
  → exact alias collision report
  → SQLite FTS5 lexical top-N candidates
  → （有 benchmark 證據時）local embedding candidates
  → curator agent 輸出 merge/keep_separate/subtype/unknown
  → SME 批准 decision record
```

處理規則：

1. 先保留 mention 的原文、所在文件和 line/heading locator；不要為了去重而刪除證據。
2. exact alias 且 scope/authority 一致時，可以自動建立 `alias_of` proposal，但仍在 MR 中批准。
3. 詞法或 embedding 相似只能建立候選對，必須比較定義、版本、scope 和 relations。
4. `merge` 批准後選定一個 canonical ID，舊 ID 寫 `superseded_by`/`alias_of`；所有引用由腳本更新並輸出可回滾 patch。
5. `keep_separate`、`subtype`、`unknown` 也要保存決策理由，避免下一次重複審查。

例：`CoroutineWorker` 和 `WorkManager` 可能存在使用關係或 subtype 關係，但不能因字面或 cosine 相似度而合成一個 entity。候選報告至少應包含：兩者定義、出現的 Android 版本、來源 authority、共同/不同 relation、相似度方法和模型版本。

### 13.3 發現與處理 Conflict

衝突檢查可由 deterministic report 先縮小範圍，再交給 SME：

1. 對相同 `predicate`、重疊 `scope` 和不同要求的 rule 做候選配對；例如同一 platform/version 下，一條要求「必須」，另一條明確說「禁止」。
2. 對 authority、有效日期、版本和文件狀態排序；較新的官方來源不會自動抹掉公司決策，因為兩者可能適用不同 scope。
3. 建立 `conflicts/<id>.yml`，列出 claim IDs、證據 locator、差異類型（scope、version、authority 或真正矛盾）、owner、due date 和 `status: open`。
4. `kb-reviewer` 只輸出「衝突摘要 + 需要的決策」，不能生成折衷 rule；`android-advisor` 必須把 open conflict 和適用範圍一併告知使用者。
5. SME 作出決定後新增 resolution/ADR，將舊 claim 標為 `superseded` 或縮小 scope；再跑 golden cases 和完整 CI。
6. 若超過 due date 未決，報告進入 escalation queue；不能因索引更新而默認其中一條為真。

可測試的回答約束：當 `get_rule` 返回 `status: open_conflict` 時，Agent 的答案必須包含 conflict ID、兩個 claim 的來源和「需要 owner 決定」；golden case 可將此列為 `must_mention` 和 `must_not_claim`。

## 14. 可直接落地的最小模板

### 14.1 Skills

`SKILL.md` 的父目錄和 `name` 必須完全一致；名稱只使用小寫字母、數字和連字號。description 要描述觸發情境和輸出，不要把整個知識庫塞進 skill。

檢索 Skill 的最小入口：

```markdown
---
name: sme-kb-retrieve
description: Retrieve cited SME knowledge, inspect related records, and report lexical or degraded retrieval mode.
---

# SME KB retrieve

1. Start with `search_knowledge` using the current `core-lexical` mode.
2. Expand only the returned evidence with `get_rule`, `get_document`, or bounded `list_related`.
3. Cite record ID, status, scope, authority, locator, and build ID.
4. State missing evidence, open conflicts, stale builds, and degraded mode; never guess.
5. Do not edit files or create a proposal.
```

維護 Skill 的最小入口：

```markdown
---
name: sme-kb-maintenance
description: Maintain the SME knowledge repository: validate sources, draft rule/entity proposals, inspect conflicts, and rebuild the read-only index.
---

# SME KB maintenance

1. Read the relevant schema and source evidence before proposing a change.
2. Never write directly to an active canonical rule; create a proposal and cite locators.
3. Run `python scripts/validate_kb.py` before handing off for review.
4. Run `python scripts/build_indexes.py` only after validation succeeds.
5. Report IDs, source paths, review status, and unresolved conflicts.
```

詳細 schema、範例和腳本放在各 Skill 的 `references/`、`assets/`、`scripts/`，利用 progressive disclosure 降低常駐上下文。若 VS Code 版本的 experimental frontmatter 字段（例如 `context: fork` 或 `allowed-tools`）未被當前 host 支持，不應把它們作為必需字段。

### 14.2 Custom Agent

下例只展示穩定的角色/工具/交接概念；實際可用工具名稱由已安裝的 VS Code/Copilot 版本決定，應在 Customizations editor 和 Agent Debug Logs 中確認：

Router（使用者通常只需選這一個）最小 frontmatter：

```markdown
---
name: sme-router
description: Route SME knowledge questions, maintenance requests, and proposal reviews to bounded workers.
user-invocable: true
tools: [agent, read]
agents: [android-advisor, kb-curator, kb-reviewer]
---

Classify the request as query, maintenance, review, mixed, clarify, or blocked.
Use only the listed transitions and at most four hops; retry a read-only worker
once. Check worker status, IDs, hashes, scope, and validation before advancing.
Never edit, approve, merge, purge, or resolve a semantic conflict.
```

```markdown
---
name: KB Curator
description: Draft evidence-backed SME knowledge proposals without changing active canonical files.
tools: [search, read, edit]
handoffs:
  - label: Send to review
    agent: KB Reviewer
    prompt: Review the proposal for authority, scope, conflicts, and source locators.
    send: false
---

You are a curator. Work only in a proposal branch or `proposals/` directory.
Every proposed claim must cite a source locator and state uncertainty.
```

`send: false` 代表交接前讓使用者確認。Router 內部可以自動執行低風險、只讀的 worker 委派；高風險寫入、purge 和 MR approval 仍必須停在同一個 human gate。若 host 忽略某個工具或 `agents` metadata，它不代表安全檢查已完成，CI、MCP、分支保護和腳本仍要執行同一套 deterministic checks。

### 14.3 建議的資料契約

所有 canonical 文件都應至少有：

```yaml
id: rule.android.example
kind: rule
title: Human-readable title
status: draft # draft|proposed|active|superseded|retired|conflict
authority: official # official|company|team|community|inferred
scope:
  platform: android
  versions: ["2.4+"]
source:
  - url: https://example.invalid/source
    locator: "heading / paragraph / line"
    retrieved_at: 2026-09-03
review_by: 2027-01-01
entities: []
relations: []
```

`inferred` 只能表示待驗證推論，不能和 `official` 共用同一 authority 優先級。Schema 應拒絕缺 source、缺 `review_by` 的 active rule；例外規則須明確寫在正文或欄位中，避免只靠標題推測。

## 15. 隨模型能力提升而成長：版本化與升級門禁

知識庫的可成長性不是「模型自動寫入更多文字」，而是每次模型升級都能重跑同一批來源、編譯器和評估，量化是否真的改善。建議把以下版本寫入 `.cache/build-manifest.json`，並在 MCP `get_build_info` 返回：

| 版本維度 | 例子 | 變更時的處理 |
|---|---|---|
| `schema_version` | `1.0.0` | 先 migration/fixture 驗證；必要時全量編譯 |
| `compiler_version` | `0.3.0` | hash 不變也可能產生不同 chunk；產生新 index，A/B 比較後切換 |
| `chunking_version` | `heading-v1` | 重新計算 locator 和 source-hit 基線 |
| `retriever_version` | `sqlite-fts5-v1` | 以同一 golden set 比較 recall、延遲和結果可解釋性 |
| `embedding_model` | `none` 或 pinned local model ID | 記錄維度、license、下載來源和 checksum；模型改動不得覆蓋舊索引 |
| `prompt/agent_version` | Git commit + agent file hash | 執行回答回歸；不要以 prompt 變更直接改 canonical facts |
| `evaluator_version` | `deterministic-v1` / `manual-review-v1` | 當前只保存可重跑指標和人工評審記錄；不保存或依賴 judge model。日後若另獲批准，才增加 evaluator model metadata |

建議採用三條發布軌：

1. **lexical fallback：** SQLite FTS5 + 人工核准 source，離線即可運行；這是 LanceDB 或 embedding 故障時的安全底線。
2. **hybrid candidate（未來）：** SQLite 控制面 + LanceDB + pinned local embedding（及可選本地 reranker），和 fallback 同時編譯，先供 benchmark/指定 Agent 使用；當前沒有 embedding，不建立此軌道。
3. **approved hybrid default：** candidate 在 golden set、來源命中、禁答條件、人工抽樣、資源成本、索引一致性和安全檢查均達標後，才更新 Agent/MCP 的預設版本；上一個完整 build 至少保留一個發布週期。

模型升級程序：

```text
pin 新模型/提示版本
  → 在相同 source commit 編譯 candidate index
  → 跑 deterministic + golden + 安全/成本 benchmark
  → 人工抽樣新增、退化和衝突案例
  → 產生比較報告與可回滾 manifest
  → SME/owner 批准
  → 更新 stable 指針；保留上一版本一個發布週期
```

當模型能力提高時，最有價值的增長順序是：更好的 evidence extraction → 更少的重複候選 → 更完整的例外/反例 → 更可靠的 query routing；不是先增加向量庫或自動生成大量 rule。新模型若只改善摘要而降低 source-hit 或增加過度泛化，應保持 candidate，不升級 stable。

## 16. 風險清單與控制措施

| 風險 | 具體表現 | 控制/監控 | 回滾或降級 |
|---|---|---|---|
| Copilot/VS Code 功能漂移 | Skill 路徑、frontmatter、MCP 字段在版本更新後改變 | 鎖定/記錄 VS Code、Copilot extension、Agent Skills spec；每月跑 smoke test | 暫退回 workspace Markdown + CLI search；保留 source 不受影響 |
| 本機 MCP 執行風險 | server 可執行任意程式、路徑穿越、過大輸出或惡意文件觸發 | stdio 只讀工具、固定 cwd、allow-list MIME/路徑、輸入 schema、limit/timeout、依賴 lock 和人工 trust review | 停用 MCP server，改用只讀腳本或直接開啟 canonical 文件 |
| 知識污染/幻覺 | inferred claim 被誤標 official；摘要脫離原文；模型捏造來源 | `authority` 強制欄位、短摘錄+locator、SME/CODEOWNERS、禁止無證據 active rule、golden `must_not_claim` | Git revert proposal/commit；由 source commit 重建 index |
| 來源過期或版權不清 | Android 文檔版本更新、URL 失效、公司文件不能外傳 | `review_by`、URL health check、license/資料分類、只存鏈接與必要短摘錄 | 標 `stale`/`retired`，保留歷史 decision；刪除未授權副本但保留 provenance |
| 索引與 source 分歧 | 半成品 SQLite、忘記重編譯、多人同時寫 index | index gitignored、hash manifest、temporary DB + atomic rename、`get_build_info` commit check | 刪除 `.cache/` 並從指定 source commit 全量重建 |
| SQLite/LanceDB 版本漂移 | SQLite 有新 chunk，但 LanceDB 向量仍來自舊正文或舊 embedding model | stable chunk IR、`chunk_id + content_sha256` 對帳、immutable build、`check_index_consistency.py`、manifest 指針切換 | 將 semantic 標為 unavailable，切 lexical fallback；刪除該 build 後重建 |
| 供應鏈/依賴風險 | Python/Node 套件漏洞、transitive dependency 變動 | lock file、pin major/minor、Dependabot/安全掃描、license review、最小 runtime image | 回到上一個 lock/容器；移除可選套件，不影響 Markdown source |
| Context/token 成本 | 全庫注入 prompt、回答變慢或忽略關鍵 evidence | query-first、top-N 上限、chunk locator、只返回摘要+按需展開、記錄 tool latency | 關閉 semantic/rerank，只保留 FTS5 和精簡 snippets |
| 多模態不一致 | host 不顯示 image content；OCR/VLM 看錯圖 | 所有資產必有 alt text/summary/Mermaid/結構化 fallback；視覺 POC 用人工抽樣 | 只返回文字摘要和檔案路徑；圖片不作唯一 rule evidence |
| Entity 錯誤合併 | 不同 API/版本被 cosine threshold 合成一個 ID | exact alias 與語義候選分開；人工 adjudication；保留 keep-separate 決策 | 恢復舊 ID/alias，重建 relations 和 index |
| Conflict 被靜默解決 | Agent 選了其中一條而沒有告知 scope | `open_conflict` 是可查詢狀態；回答約束和 golden case 必須提及 conflict | 標記 unresolved，暫停該 query 的確定性建議 |
| Router 錯誤委派或循環 | 模型把 maintenance 當 query、呼叫未授權 worker、反覆重試或遺失 context | intent enum、固定 transition table、`agents` allow-list、最多 4 hops/每 stage 1 retry、visited-agent/handoff hash 檢查；保留 Router failure trace | `clarify`/`blocked`，停止委派；必要時由工程師修正 Agent contract，不擴大權限 |
| 人工審查瓶頸 | proposals 排隊太多，規則長期停在 draft | 依風險/authority 分級；每週 review queue；統計 lead time 和 stale count | 限制 AI 只做報告，不擴大自動寫入範圍 |
| 評估被單一分數誤導 | 外部 evaluator 分數上升但 source-hit/人工品質下降 | deterministic 指標 + golden + 人工抽樣；當前不使用 LLM evaluator，也不設跨模型通用 0.95 | 保持 stable 版本，撤回 candidate；分析失敗案例再調整 |

## 17. 最終調整方向總結

| 主題 | 原稿可能的方向 | 調整後的可執行方向 | 調整理由/決策影響 |
|---|---|---|---|
| Canonical source | Git、Markdown、YAML、Mermaid 和多個索引都像同一層真相 | 只把 Git 文件當 SSOT；SQLite/LanceDB/圖報告全部可重建 | 降低同步衝突，支持 diff、MR、回滾 |
| Copilot instructions | 把大量 SME 內容放常駐提示 | 全局 instructions 只放查詢、引用、信任和安全規則；事實按需檢索 | 減少 token 壓力，避免 instruction 與 canonical rule 分叉 |
| Skills | 用一個 SKILL.md 承載全部知識 | 使用兩個短 core Skills：`sme-kb-retrieve` 定義證據檢索，`sme-kb-maintenance` 定義 proposal/lifecycle 流程；資料放 `knowledge/`，reference/script 漸進載入 | 符合 Agent Skills progressive disclosure，分離讀取和寫入風險，避免 Skill 膨脹 |
| Custom agents | 一個 Agent 同時搜尋、編輯、發布 | `sme-router` 作日常入口，advisor/curator/reviewer 分角色；Router 只按 allow-list 委派並檢查流程；handoff/MR 要人確認 | 最小權限、可審計，避免無人監管修改；Router 不是 semantic authority |
| MCP | 暴露通用檔案/SQL/寫入工具 | 只讀、typed、受限的 `search/get/list/build-info` 工具 | 降低本機任意程式和 prompt injection 風險 |
| Search | 第一天即部署向量/圖資料庫，或只靠單一檢索器 | 當前只發布 SQLite FTS5 `core-lexical`；保留 LanceDB/embedding provider seam，未來以同一 MCP `lexical/semantic/hybrid` 契約做 candidate benchmark，故障可退回 lexical | 先交付可離線、可回滾的低複雜度底座；只有 provider、品質、資源和一致性 gate 全部通過才啟用 hybrid |
| Graph | Kuzu 作核心，Leiden 自動重構目錄 | 顯式 relations 編譯到 SQLite；Leiden 僅作離線 audit；Kuzu 不列基線 | Kuzu repo archived；分群不能代替業務邊界判斷 |
| 多模態 | 存圖即可跨模態理解 | 原圖 + sidecar alt/summary/provenance；MCP image 只是有條件增強 | 協議支持不代表 Copilot host/model 必然視覺推理 |
| AI 抽取 | 批量自動生成並寫入 entities/rules | AI 只產生 proposal，帶原文、locator、不確定性；SME 批准後合入 | 控制 hallucination、版權和錯誤累積 |
| Entity resolution | 用相似度閾值自動合併 | normalization → alias → FTS/embedding 候選 → adjudication → MR | cosine 相似只表示候選，不是語義真相 |
| Conflict | 自動選最新/最高分規則 | 保存 open conflict、owner、due date 和 scope；回答必須揭示未決狀態 | 防止 Agent 把矛盾隱藏成單一錯誤答案 |
| Quality | Ragas/單一閾值作硬門禁 | deterministic schema/link/source checks + golden/人工基線；當前不使用 Ragas 或 LLM judge | 評估分數受 judge、prompt 和資料集影響；不能代替來源證據 |
| 增長 | 模型越強，知識自動幾何級增長 | 版本化 schema/compiler/retriever/model/prompt；candidate → benchmark → approved default | 把「模型能力」轉化為可證明、可回滾的質量改進 |
| 性能 | 預先承諾 10–50ms 等固定數字 | content hash 增量編譯；以本機文件量和硬件 benchmark 設定 SLO | 避免未測量的性能承諾 |

## 18. 實施前的環境核對清單

在開始階段 0 前，建議由你或平台管理員把下列結果附到 issue/README。這些檢查不需要模型 API：

```bash
python3 --version
python3 -m pip --version
python3 -m venv --help
node --version
npm --version
git --version
python3 -c "import sqlite3; print(sqlite3.sqlite_version); c=sqlite3.connect(':memory:'); print(c.execute(\"select sqlite_compileoption_used('ENABLE_FTS5')\").fetchone()[0])"
python3 -c "import lancedb; print('lancedb import: ok')"
test -f .vscode/mcp.json || true
test -d .github/skills || true
test -d .github/agents || true
npx --yes @mermaid-js/mermaid-cli --version   # 僅在允許下載/已有 npm cache 時執行
```

依賴安裝建議（以 Python 內建 `venv` + `pip` 作為跨環境基線）：

```bash
python3 -m venv .venv
source .venv/bin/activate                 # Windows PowerShell: .venv\\Scripts\\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip install -r requirements-dev.txt
# `fastembed` 只在選中的模型出現在官方 supported list 時二選一替代 sentence-transformers
# Mermaid CI 另在固定 Node image/package 中安裝並鎖定版本
npm install --save-dev @mermaid-js/mermaid-cli
```

`mcp` SDK 的 major version、LanceDB、以及 embedding provider 必須和 server/compiler 一起鎖定；上面的 `requirements*.txt` 只是可移植範例，不是公司批准的最終 lock。團隊也可以用 Poetry、uv、Conda 或其他已批准工具管理同一份 `pyproject.toml`/依賴鎖定；切換管理器不應改變 `python scripts/...` 的 compiler 入口。`fastembed` 只是第一個 CPU-friendly POC 候選，也可以由 `sentence-transformers` 替代，但兩者不應同時成為 baseline。先在隔離環境建立 lock、下載並核對模型 checksum、跑 license/security scan，再提交 `pyproject.toml` 和對應 lock file。若公司不能使用本地 embedding 或 MCP，仍可保留 SQLite compiler 和 Skill/Agent 文件，以 lexical-only/CLI search 啟動。

### 18.1 依賴的授權與服務邊界補充

| 類別 | 代表依賴 | 授權/服務注意 |
|---|---|---|
| 系統與 runtime | Git（GPLv2）、Python（PSF）、Node.js（MIT）、SQLite（public domain） | 這些是本機工具/執行時；GitHub、VS Code、Copilot plan/企業政策另行管理，不等同於開源授權 |
| Python baseline | `mcp`（官方 SDK，MIT）、Pydantic（MIT）、PyYAML（MIT）、jsonschema（MIT）、pytest（MIT）、ruff（MIT/Apache-2.0） | 以 lock 中的確切版本和 transitive license 做公司法務審查；PyYAML 只用 `safe_load` |
| Node/diagram | `@mermaid-js/mermaid-cli`（MIT）及其 transitive packages | CI runner 需允許 npm registry 或使用已審批 container；固定 lock，避免每次 MR 下載浮動依賴 |
| 可選檢索 | LanceDB（Apache-2.0）、fastembed/sentence-transformers（按 pinned model/package license） | 模型權重 license、下載來源、向量是否含敏感文本要單獨審查；不要因套件開源就假定模型可商用 |
| 可選圖分析 | `leidenalg`/`python-igraph` 的 license 以實際版本為準 | GPL/第三方 license 可能影響分發；圖分析只在離線受控環境執行；Ragas/LLM evaluator 不列入當前依賴 |
| 外部服務 | VS Code GitHub Copilot（互動式）、GitLab CI/CD、npm registry（僅 Mermaid CI） | 需確認企業資料保留、網絡出口、job token、費用、DPA/版權政策；本方案當前基線不依賴外部 LLM API、Ragas evaluator 或模型服務 |

## 19. 研究來源與可重查證據

以下是本報告截至 **2026-09-04** 的主要來源；產品文檔和套件版本會變動，正式上線前應按當時版本重查。來源只用於驗證能力邊界，canonical knowledge 仍需保存自己的 locator 和 review date。

### VS Code、GitHub Copilot、Agent Skills

- [GitHub Copilot repository custom instructions](https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions-for-github-copilot)：`.github/copilot-instructions.md`、path-specific instructions 和避免衝突的提示。
- [VS Code agent customization overview](https://code.visualstudio.com/docs/agent-customization/overview)：workspace/user customization 發現範圍和設定入口。
- [VS Code Agent Skills](https://code.visualstudio.com/docs/agent-customization/agent-skills)：`SKILL.md`、frontmatter、目錄命名、progressive disclosure 和技能位置。
- [Agent Skills specification](https://agentskills.io/specification)：標準字段、可選 `references/`、`scripts/`、`assets/` 和 validator。
- [VS Code Custom Agents](https://code.visualstudio.com/docs/agent-customization/custom-agents)：`.agent.md`、工具限制、subagents 和 handoffs。
- [VS Code MCP servers](https://code.visualstudio.com/docs/agent-customization/mcp-servers)：`.vscode/mcp.json`、stdio server、信任和 sandbox 注意事項。
- [VS Code Subagents](https://code.visualstudio.com/docs/agents/run/subagents)：stateless focused delegation、`agents` allow-list、nested-depth/termination 和 coordinator/worker pattern。
- [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)：routing、簡單 composable workflows、tool ground truth、human checkpoints 和 stopping conditions。
- [LangChain multi-agent patterns](https://docs.langchain.com/oss/python/langchain/multi-agent)：Router、Subagents、Handoffs、Skills 的差異和 context/call trade-offs。
- [AutoGen SelectorGroupChat](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/selector-group-chat.html)：候選 Agent 限制、自訂 selector、termination 和 human feedback 概念。
- [OpenAI Agents SDK guardrails](https://openai.github.io/openai-agents-python/guardrails/)：input/output/tool tripwire 和 fail-closed guardrail 概念（僅作研究，不是本項目依賴）。
- [VS Code agent trust and safety](https://code.visualstudio.com/docs/agents/concepts/trust-and-safety)：approval、sandbox、trust boundary 和 review-before-commit；說明 prompt 不是安全邊界。

### MCP、儲存和搜尋

- [MCP tools specification（2026-07-28）](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)：工具呼叫、structured output、text/image/resource content 和人為確認建議。
- [Official MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)：Python 3.10+、v2 stable line、stdio/HTTP transport 和 SDK 版本邊界。
- [LanceDB documentation](https://lancedb.com/docs/) / [quickstart](https://lancedb.com/docs/quickstart)：embedded local library、metadata/vector/multimodal column 和連線方式。
- [LanceDB Full-Text Search](https://docs.lancedb.com/search/full-text-search.md) / [FTS index](https://docs.lancedb.com/indexing/fts-index.md)：不需 embedding 的 BM25 FTS、索引建立與 `optimize()` 更新行為。
- [LanceDB Hybrid Search](https://docs.lancedb.com/search/hybrid-search.md) / [RRF reranker](https://docs.lancedb.com/reranking/rrf.md)：vector + FTS hybrid、explicit vector/text query、內置 RRF 融合。
- [SQLite about](https://www.sqlite.org/about.html)：in-process、serverless、single-file、transaction/index 特性。
- [Kuzu repository](https://github.com/kuzudb/kuzu)：封存狀態和歷史 embedded graph 功能；因此不列為新基線。

### 評估、圖表和 Android domain evidence

- [Ragas documentation](https://docs.ragas.io/en/stable/) / [available metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/) / [Faithfulness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/)：評估 loop、context/faithfulness/relevance 和 evaluator 依賴。
- [Mermaid introduction](https://mermaid.js.org/intro/) / [Mermaid CLI](https://mermaid.js.org/config/mermaidCLI.html)：文字定義、diffable diagram 和 CLI render。
- [Android app architecture](https://developer.android.com/topic/architecture)：layering、single source of truth、UDF、repository 邊界。
- [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)：`repeatOnLifecycle` 的 UI collection 警告及 `lifecycle-runtime-ktx:2.4.0+` 例子。

本機環境探測和原始主張對照保存在：

- [verified-sources.md](<./docs/verified-sources.md>)

## 20. 結論與下一步

推薦的最小可行產品（MVP-Lexical）是：20–50 份 Android 高價值來源、20–40 條人工核准 rule/entity、合法 Skills 和三個角色 Agent、SQLite 控制面、`DisabledEmbeddingProvider`、LanceDB provider seam（不建立 vector table）、只讀 MCP、CI schema/dead-link/Mermaid/索引一致性 checks，以及 20 個 golden cases。它可以在目前沒有 embedding 的環境交付；日後只需加入 pinned provider 和新的 LanceDB build，便能在相同 golden cases 上量測 hybrid 是否改善召回，不需要修改 canonical knowledge 或 MCP 契約。

開始實施時，按這個順序執行：

1. 確認 Copilot/VS Code 功能、企業資料政策和 MCP trust；把版本與結果記錄在 issue。
2. 建立 schema、sources 和第一批 rules，先讓 deterministic CI 通過。
3. 加入 advisor/curator/reviewer customization，驗證每個答案都能返回 rule ID/source 或明確說「無證據」。
4. 加入 SQLite compiler、`DisabledEmbeddingProvider`、LanceDB adapter seam 和只讀 MCP；先以 lexical-only 完成 source-hit、延遲、治理和人工品質基線。
5. 未來取得模型批准後，再建立獨立 LanceDB vector build，對 lexical/hybrid 測量 source-hit、recall、延遲、RAM 和人工品質；通過一致性與資源門禁後才把 hybrid 設為預設。

這樣的設計可以隨模型能力上升而受益，但不把模型輸出當成真相，也不把新基礎設施當成知識品質本身。知識品質的核心增長來源是：更好的來源、明確的 schema/provenance、可重複的評估、及時的 SME 決策，以及每次變更都能被審查和回滾。

## 21. 研究附錄：雙索引版架構修改計劃（非規範）

> 本節至文檔結尾保留先前研究和比較，僅作背景及證據。工程師不得從本節選擇實作方案；當前唯一規範是 [SME 知識庫工程實作規格](<./SME-知識庫工程實作規格.md>)。其中的 future/optional、替代方案和 Open Questions 不會覆蓋規範文件的現行決定。

### 21.1 Requirement Breakdown

| 需求 | 架構回應 | 非目標 |
|---|---|---|
| 同時使用 SQLite 和 LanceDB | SQLite 控制面 + LanceDB 語義面；由一個本機 Python gateway 統一讀取 | 不把兩個資料庫當成兩份 canonical source |
| VS Code GitHub Copilot、非 API | Copilot 只經 Skills/Router/Custom Agents/MCP 呼叫 retrieval；當前使用 `DisabledEmbeddingProvider`，未來才由本地 provider 執行 embedding | 不在 GitLab CI 直接調用 Copilot Chat API 或任何 LLM API |
| 可維護、複雜度低 | 一個 package、一個 stdio MCP process、兩個 embedded stores、immutable build snapshots | 不引入常駐服務、Kuzu、Neo4j 或微服務網絡 |
| 質量隨模型變強 | embedding/prompt/agent/compiler 都版本化；candidate 與 stable 可 A/B、可回滾 | 不允許模型直接合入 active rule 或自動合併 entity |
| 供其他 Agent 使用 | MCP 返回 hybrid evidence、IDs、scope、provenance 和 retrieval mode；工具契約穩定 | 不要求其他 Agent 了解 LanceDB table 或內部 SQL |

### 21.2 Technical Objectives and Strategy

成熟化的核心不是增加資料庫數量，而是建立明確的控制面和資料面：

- **Control plane（SQLite）：** 決定哪一個 document/chunk/entity/rule 是 active、適用哪個 scope、誰審批、來源在哪裡、哪個 build 是 current。
- **Semantic plane（LanceDB）：** 對同一批 stable chunks 保存向量和 ANN 索引，支援語義召回、重複候選和未來 image/text embedding；不保存新的事實。
- **Shared kernel（stable chunk IR）：** parser、normalizer、chunker 只執行一次；兩個 store 都使用相同 `chunk_id`、`content_sha256`、`locator` 和 `modality`。
- **Facade（retrieval gateway）：** MCP 和 Agents 只看到 `search_knowledge` 等 typed tools；gateway 先查兩個索引，再以 SQLite 核對 authority/status/provenance。

MCP 是跨 Agent 的穩定 interoperability boundary：其他支援 MCP 的 Agent 只需理解工具輸入/輸出契約即可使用知識，不需要安裝 LanceDB。若某個 Agent 不支援 MCP，仍可讀取 Git canonical files 或呼叫同一個 CLI facade；不要為每個 Agent 再寫一套檢索邏輯。

推薦的 hybrid query sequence：

```text
query + filters
    ├─ SQLite FTS5 → lexical candidates
    └─ local embedding → LanceDB semantic candidates
              ↓
       chunk_id merge + RRF/weighted fusion
              ↓
       SQLite status/scope/provenance verification
              ↓
       MCP evidence response（或 lexical fallback + degraded flag）
```

### 21.3 Involved Files

| 變更區域 | 直接受影響內容 | 依賴/邊界 |
|---|---|---|
| `knowledge/` | entities、rules、sources、proposals、conflicts、assets、snippets | canonical；不依賴 SQLite/LanceDB API |
| `schemas/` | common/entity/rule/relation/asset/snippet/proposal/manifest schema | deterministic validator 的輸入契約 |
| `src/sme_kb/compiler.py` | stable chunk IR、hash、staging build、publish pointer | 只讀 source；呼叫兩個 store adapter |
| `src/sme_kb/stores/sqlite_store.py` | FTS5、metadata、relations、provenance、index_builds | Python `sqlite3`；控制面唯一過濾來源 |
| `src/sme_kb/stores/lancedb_store.py` | vector table、metadata、upsert/search、model/dimension metadata | `lancedb` + 單一本地 embedding provider；`disabled` profile 不呼叫 vector path |
| `src/sme_kb/retrieval.py` | lexical/semantic/hybrid、fusion、fallback | 不讓 Agent 直接 import store internals；根據 provider capability 選擇 lexical-only 或 hybrid |
| `src/sme_kb/mcp_server.py` | typed read-only tools、resource links、輸出上限 | 官方 `mcp` SDK；stdio；路徑和 MIME policy |
| `configs/` | retrieval weights、model ID、dimension、policy、limits | 版本控制；不放 secret |
| `scripts/` | validate/build/consistency/benchmark/maintenance report | CLI facade；可被 CI 和本機手動執行 |
| `tests/` | contract、retrieval、consistency、security、golden regression | 不依賴 Copilot 或 LLM API；當前使用 fixture/DisabledEmbeddingProvider，未來模型測試另行加入 |

### 21.4 Layer Topology and Shared Kernel Placement

```text
knowledge files (canonical)
        ↓
contracts + parser/normalizer/chunker (shared kernel, deterministic)
        ├── SQLite adapter (control plane)
        └── LanceDB adapter (semantic plane)
        ↓
retrieval facade (hybrid/fallback policy)
        ↓
MCP interface (typed tools/resources)
        ↓
VS Code Skills / Custom Agents / other Agents
```

`contracts.py`、ID/hash、scope、authority 和 provenance 屬於 shared kernel；它們不可 import MCP、LanceDB 或 Copilot。`stores/` 是 infrastructure adapters；`retrieval.py` 是 application orchestration；`mcp_server.py` 是 interface boundary。這個分層可避免日後把 vector-specific field 滲透到 canonical Markdown 或 Agent prompt。

### 21.5 Detailed Per-File Plan

| 文件/目錄 | 實施內容 | 完成條件 |
|---|---|---|
| `schemas/*.schema.json` | 依 9.5 清單建立 common/source/entity/rule/relation/process/decision/asset/snippet/proposal/conflict/taxonomy/chunk-ir/manifest，並用 `$ref` 共用欄位；active rule/snippet 強制 status/authority/scope/source/review_by | valid/invalid fixtures 和 GitLab validator 通過；缺字段、未知枚舉、snippet 多於一個 fenced body 或缺安全 metadata 時 fail |
| `src/sme_kb/compiler.py` | parse、normalize、chunk、hash、建立 staging build；不在此處調用 Copilot/LLM | 相同 source commit 產生相同 IR/hash；重跑結果可比較 |
| `sqlite_store.py` | 建立 documents/chunks/relations/index_builds/chunk_vectors；FTS5 查詢和狀態過濾 | 可由 `chunk_id` 取回完整 provenance；不可接受任意 SQL |
| `lancedb_store.py` | 建立以 model ID/dimension 命名的衍生 table；保存 chunk hash、modality、build ID | vector search 結果可被 SQLite 對帳；model 改變不覆蓋舊 table；disabled 時不建立空向量表 |
| `retrieval.py` | lexical/semantic/hybrid mode；RRF/權重配置；控制 semantic unavailable fallback | golden set 可比較兩種 mode；結果含 score breakdown、provider capability 和 degraded flag |
| `mcp_server.py` | 只暴露 search/get/list/build-info/asset/snippet；輸出 schema、limit、timeout、safe path；snippet 只返回純文字 | VS Code 可發現工具；path traversal、過大輸出、unknown ID 和 snippet execution attempt 測試通過 |
| `scripts/check_index_consistency.py` | missing/orphan/hash/model/manifest mismatch 報告 | mismatch 令 build 不發布，或明確發布 lexical-only |
| `.cache/` | 以 build ID 保存 immutable SQLite/LanceDB；`current.json` 指針切換 | MCP 不讀半成品；刪除 cache 可完整重建 |
| `.gitlab-ci.yml` | schema/dead-link/Mermaid、deterministic build、consistency、lexical regression | MR 結果可重現；不安裝或呼叫外部 LLM API |

### 21.6 Old → New Mapping

| 舊方向 | 雙索引成熟版 | 遷移方式 |
|---|---|---|
| `scripts/compile_index.py` 只建 SQLite | `build_indexes.py` 建 stable IR，再同時產生 SQLite/LanceDB snapshot | 先保留舊 CLI 名稱作 thin wrapper；確認新 build 一致後移除舊實作 |
| `.cache/index.sqlite` 單一可變文件 | `.cache/builds/<build-id>/{sqlite,lancedb}` + `current.json` | 由 source commit 全量建第一個 build；不搬移 canonical files |
| `knowledge/concepts/` | `knowledge/entities/`，另加 proposals/conflicts/taxonomy | 保留 entity IDs；只改目錄和 references，不重寫定義 |
| MCP 直接查 SQLite/檔案 | MCP → retrieval facade → SQLite + LanceDB | 保持工具名穩定，內部 store 可替換 |
| LanceDB 作為未來選項 | hybrid profile 的語義面；仍可用 lexical fallback | 先選一個 local embedding，建立 candidate；通過 benchmark 才設預設 |
| 固定相似度閾值自動合併 | 相似度只產生候選；SQLite scope/authority + SME adjudication | 舊自動 merge 規則改成 proposal/decision record |

### 21.7 Cohesion/Facade Plan

- `src/sme_kb/retrieval.py` 是唯一的 retrieval facade；MCP、CLI、benchmark 和未來其他 Agent 都呼叫它，不直接 import `sqlite_store` 或 `lancedb_store`。
- `src/sme_kb/contracts.py` 是穩定 DTO/typed response 邊界；MCP output schema 和 golden/manual evaluator 共享同一契約，當前不依賴 LLM judge。
- `scripts/` 只做命令列適配和報告，不複製 compiler/search 邏輯；避免日後 CLI、MCP、CI 各自產生不同答案。
- embedding provider 以介面隔離；`fastembed`/`sentence-transformers` 是可替換 adapter，而非散落在 rule/Agent 文件中的固定實作。即使目前不能使用模型，也要保留相同介面和 `DisabledEmbeddingProvider`，讓 compiler/retrieval 不需要在日後重新改造。

### 21.8 Risk/Dependency Assessment

引入 LanceDB 後新增的主要風險是本地模型下載、向量維度/模型版本漂移、PyArrow/平台 wheel、CPU/RAM、索引一致性和模型權重 license。這些風險由 `models.yml`、lock/checksum、staging build、consistency checker 和 lexical fallback 控制。SQLite 仍是必要控制面；LanceDB 可暫停，但不能用一個沒有 provenance 的向量結果代替 SQLite。

未來啟用語義層時建議只選一個 embedding provider。先做文本向量，不同時做 image embedding、reranker、Ragas 和 Leiden；每增加一個組件都要有 benchmark、license review、資源預算和 rollback。當前不安裝或載入 provider，避免把不可用組件變成 runtime 依賴。

### 21.9 Validation and Rollout Gates

| Gate | 必須通過的檢查 | 不通過時 |
|---|---|---|
| G0 環境 | `sqlite3` FTS5、`import lancedb`、`EmbeddingProvider` contract test、MCP stdio 啟動；只有選 `hybrid` 時才要求 embedding model smoke test | 沒有模型時使用 `DisabledEmbeddingProvider` + lexical-only；記錄 blocker，不改 canonical schema |
| G1 Source | schema、ID、source locator、review_by、dead-link、Mermaid | MR 不合入 |
| G2 Build | IR deterministic、SQLite build；若 provider `ready` 才增加 LanceDB build；manifest 對帳 | core-lexical 只發布 SQLite snapshot；hybrid 任一索引失敗時不切換 `current.json`，保留 lexical build/上一完整 build |
| G3 Retrieval | golden source-hit、recall@k、duplicate-candidate precision、p95/RAM/磁碟 | hybrid 保持 candidate；MCP 預設 lexical fallback |
| G4 Copilot | Skill/Agent discovery、MCP tool discovery、答案引用 rule/source、open conflict 行為 | 關閉 semantic 或 MCP，退回 workspace files/CLI |
| G5 Governance | CODEOWNERS、proposal-only edit、rollback/rebuild 演練、license/security review | 不發布 stable；修正治理或依賴後重跑 |

### 21.10 Assumptions/Open Questions

- 已知目前環境不能使用 embedding model；這不是 MVP blocker，`DisabledEmbeddingProvider` + `core-lexical` 是當前正式 profile。日後若政策允許模型權重，再以相同 provider interface 建立獨立 hybrid build。
- 需要確認 Android/公司知識是否允許送入本地 embedding；即使不出網，本地向量仍可能屬於敏感衍生資料，需要同樣的存取和刪除政策。
- 需要用實際 20–50 份文件測量 hybrid 是否改善 recall、duplicate candidate quality 和 p95 latency；不能預先承諾固定閾值。
- 需要確認 VS Code/Copilot 版本是否接受本機 MCP 的 image content；多模態 fallback 必須先保留。
- 需要決定 build artifact 是否只在本機生成，或在受控 CI 產生後分發；個人庫建議本機生成，內部共享庫才評估 artifact cache/簽名。

## 22. 本地 Embedding model 選型

### 22.1 直接結論

對目前的 Android SME 知識庫，當前正式 baseline 是：

> **`DisabledEmbeddingProvider` + SQLite FTS5 + `core-lexical`**

embedding 部分目前只保留 provider interface 和模型選型記錄，不安裝或載入模型。未來如果可以啟用語義層，`BAAI/bge-m3` + `sentence-transformers` + LanceDB cosine/dense index 仍是 quality candidate：官方模型卡列出 100+ 語言、1024 維、8192 sequence length、MIT license，並建議 RAG 使用 hybrid retrieval 和後置 reranking。這些資料是候選依據，不是目前可部署的 runtime 決策。

未來模型啟用也不是無條件的「最佳分數」宣稱：模型卡和公開 benchmark 不能代替你們的 Android/公司語料測試。應以 BGE-M3 作 quality candidate，和 E5-base/small 做資源對照；通過自己的 golden set 後才把它標為 approved hybrid default。

BGE-M3 是文字模型，不是 image encoder。未來語義層啟用時，圖片檢索可先使用圖片的 alt text、summary、OCR（若已人工核實）和 Mermaid/結構化節點，將這些文字和其他 chunk 一起嵌入；當前 `core-lexical` 則只依賴這些文字欄位做 FTS5。只有在確實需要「文字 query 找原圖/圖片 query 找文字」時，才另行評估 CLIP/SigLIP 類 image-text model；不要把它和 BGE-M3 混成一個向量空間，也不要因此提前增加第二個模型 runtime。

### 22.2 候選比較

| 模型 | 語言/特色 | 維度與最大序列 | 觀察到的權重下載量* | License | 建議 |
|---|---|---:|---:|---|---|
| `BAAI/bge-m3` | 100+ 語言；dense、sparse、multi-vector；適合長技術段落 | 1024 / 8192 | 約 2.27 GB | MIT | **品質優先首選**；Mac 16 GB RAM 以上較合理，使用 `sentence-transformers` 或 FlagEmbedding dense mode |
| `intfloat/multilingual-e5-base` | 多語言；模型卡要求 retrieval input 使用 `query:` / `passage:` prefix | 768 / 512 | 約 1.11 GB | MIT | **平衡首選**；若 BGE-M3 CPU/RAM/啟動成本過高，先用它 |
| `intfloat/multilingual-e5-small` | 多語言；同樣需要 `query:` / `passage:` prefix | 384 / 512 | 約 0.47 GB | MIT | **低資源 smoke test/fallback**；速度和容量較好，但預期品質低於 base/BGE-M3 |
| `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` | 384 維、多語言句子相似度模型；官方 card 的有效 max sequence 較短 | 384 / 約 128 effective in sentence-transformers config | 約 0.47 GB | Apache-2.0 | 只作兼容性 fallback；不作 Android technical retrieval 的主模型 |

\* 下載量是 2026-09-03 從 Hugging Face model artifact 的 HTTP `Content-Length` 觀察值，不等於實際推理 RAM；量化、框架、batch size 和 Apple Silicon backend 都會影響記憶體。

如果你未提供硬件規格，我會這樣決策：

| 環境 | 預設模型 | 原因 |
|---|---|---|
| Apple Silicon / 16 GB RAM 以上 | BGE-M3 | 中文/英文技術和較長 chunk 的品質優先；可用 fp16/量化 POC 減少成本 |
| 8–16 GB RAM，CPU-first 或需要較快啟動 | multilingual-e5-base | 768 維和約 1.1 GB 權重，品質/資源較平衡 |
| 8 GB RAM、只做小型試點或 CI smoke | multilingual-e5-small | 384 維、約 0.47 GB；先驗證 pipeline，再升級模型 |
| 不能下載模型權重或公司不批本地 inference | 不啟用 semantic plane | LanceDB 保留 schema/目錄但標 disabled；MCP 走 SQLite FTS5 lexical-only |

### 22.3 Runtime 和資料契約

Embedding provider 只應有一個穩定介面，實作可以更換：

| 契約 | BGE-M3 | multilingual-E5 |
|---|---|---|
| document input | `title + aliases + heading + body + scope keywords`；不要把 URL/status 當語義內容 | 同左，但保存的 passage string 要加 `passage: ` prefix |
| query input | 原始 query（可加短 domain/context，但不把 filters 混入正文） | 必須加 `query: ` prefix；官方 card 也要求非英文輸入使用 prefix |
| output | 固定 `float32` 或經核准的 `float16` → L2 normalize；dimension 1024 | 同左；dimension 768/384 依模型 |
| distance | LanceDB cosine；metric 寫入 manifest | 同左；不可混用不同 dimension/table |
| version key | model ID + exact revision + provider version + dimension | 同左；任何改動建立新 table/build |

#### 22.3.1 Interface-first 與空實現

目前環境不能使用 embedding 時，仍然建立同一個 `EmbeddingProvider` 抽象，但注入 `DisabledEmbeddingProvider`（Null Object）。它的責任是明確回報「語義能力未啟用」，而不是模擬一個模型：

```python
from dataclasses import dataclass
from typing import Protocol, Sequence


class EmbeddingUnavailable(RuntimeError):
    """Raised if a disabled/unavailable provider is called by a semantic path."""


@dataclass(frozen=True)
class EmbeddingInfo:
    provider: str
    model_id: str | None
    revision: str | None
    dimension: int | None
    status: str  # ready | disabled | unavailable


class EmbeddingProvider(Protocol):
    @property
    def info(self) -> EmbeddingInfo: ...

    def embed_documents(self, texts: Sequence[str]) -> list[list[float]]: ...

    def embed_query(self, text: str) -> list[float]: ...


class DisabledEmbeddingProvider:
    info = EmbeddingInfo(
        provider="disabled",
        model_id=None,
        revision=None,
        dimension=None,
        status="disabled",
    )

    def embed_documents(self, texts: Sequence[str]) -> list[list[float]]:
        raise EmbeddingUnavailable("embedding model is not configured")

    def embed_query(self, text: str) -> list[float]:
        raise EmbeddingUnavailable("embedding model is not configured")
```

實作規則：

- `core-lexical` 建置只注入 `DisabledEmbeddingProvider`，建立 SQLite FTS5；compiler 不載入模型，也不建立空的 LanceDB vector table。
- `hybrid` 建置只接受 `status: ready` 的真實 provider；模型載入失敗時不發布半成品，或明確發布 `lexical-only + degraded` build。
- `retrieval.py` 先檢查 `provider.info.status`，只有 `ready` 才呼叫 `embed_query`；誤入 semantic path 時讓 `EmbeddingUnavailable` 立即暴露配置錯誤。
- 絕不返回零向量、隨機向量或空向量作為假實現。這些值可能通過型別檢查，卻會產生錯誤的 LanceDB 分數和不可解釋的召回結果。

配置可以先固定為：

```yaml
embedding:
  provider: disabled
  status: disabled
  model_id: null
  revision: null
  dimension: null
```

日後改成 `sentence-transformers` 或其他 adapter 時，只需替換 provider 和模型 metadata；`compiler.py`、`retrieval.py`、MCP 工具契約以及 canonical `knowledge/` 不需要改寫。這是 dependency inversion 的實際收益，而不是把未使用的套件先加入 runtime。

Chunk 初始建議先用 heading/段落邊界、約 250–500 tokens 的候選範圍，保留 `parent_document_id` 和 locator；這是待 benchmark 的起點，不是固定真理。E5 最大序列較短，超長段落應在 compiler 內分割或摘要，而不是依賴模型靜默截斷。程式碼/API identifier 仍由 SQLite FTS5 負責 exact matching，不能只依賴 embedding。

### 22.4 建置和執行方式

1. **Build time：** `build_indexes.py` 讀取 source → stable chunk IR；`core-lexical` 使用 `DisabledEmbeddingProvider` 並跳過 vector build，只有 provider `ready` 的 `hybrid` profile 才在 staging 載入模型、批量產生 vectors、寫入 LanceDB；完成 hash/model/dimension 對帳後發布。
2. **MCP startup：** 只讀 `.cache/current.json`；若預設 retrieval mode 是 hybrid，啟動時載入同一個 pinned model 一次並 cache，不要每次 query 重新載入。
3. **Query time：** 只有 provider `ready` 時才在本機產生 query embedding；不出網、不使用 Copilot API key。`disabled` profile 直接走 lexical-only；若 hybrid 模型載入失敗，返回 `retrieval_mode: lexical_fallback` 和 degraded flag。
4. **Offline policy：** 模型首次下載在受控 build/setup 階段完成；runtime 設 `local_files_only`/等價 offline mode，避免 MCP 被動觸發網絡下載。
5. **Reproducibility：** `models.yml` 保存 Hugging Face model ID、exact revision、dimension、pooling/normalization、provider、checksum 和模型 license；不要只保存 `latest`。

### 22.5 Provider 選擇的修正

目前不建議把 `fastembed` 當成所有模型的預設 runtime。FastEmbed 的官方 supported-models 頁面列出其支援的模型、維度、license 和大小；截至本研究日期，頁面可見 multilingual MiniLM 和 multilingual-E5-large，但不能據此推定 BGE-M3、E5-base 或 E5-small 都可直接使用。若選 BGE-M3/E5-base，先用 `sentence-transformers`/官方 FlagEmbedding 路徑；若選 fastembed supported model，才把它當作替代 provider，並在 G0 smoke test 驗證向量維度和結果。

來源：

- [BAAI/bge-m3 model card](https://huggingface.co/BAAI/bge-m3)：100+ languages、1024/8192、MIT、dense/sparse/multi-vector 和 hybrid/rerank 建議。
- [intfloat/multilingual-e5-base model card](https://huggingface.co/intfloat/multilingual-e5-base)：多語言、768 維、512 max、MIT，以及 `query:`/`passage:` prefix 要求。
- [intfloat/multilingual-e5-small model card](https://huggingface.co/intfloat/multilingual-e5-small)：多語言、384 維、512 max、MIT，以及相同 prefix 要求。
- [paraphrase-multilingual-MiniLM-L12-v2 model card](https://huggingface.co/sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2)：384 維、Apache-2.0 和 sentence-transformers pooling/config。
- [FastEmbed supported models](https://qdrant.github.io/fastembed/examples/Supported_Models/)：模型維度、描述、license 和下載大小清單。

## 23. 沒有本地 Embedding 時的降級架構

### 23.1 影響有多大

沒有 embedding **不會阻塞知識庫、Skills、Custom Agents、MCP、provenance 或規則治理**；受影響的是「語義召回」和依賴語義相似度的維護工作：

| 能力 | 沒有 embedding 的影響 | 可否用 SQLite/規則補足 |
|---|---|---:|
| Android API/class/函數精確搜尋 | 低；`WorkManager`、`StateFlow` 等 exact token 由 FTS5 很適合 | 可以，加入 aliases、keywords、normalized identifier |
| 中文問題找英文官方文件、同義改寫 | 高；詞面不同時容易漏召回 | 部分可以，靠雙語 aliases、query expansion 和人工 taxonomy |
| Entity duplicate candidate | 中至高；只能先找到 exact/詞法相似候選 | 部分可以；候選需人工 adjudication，不能自動合併 |
| Rule/conflict/source 回答 | 低至中；只要查到正確 rule，回答與治理仍可運作 | 可以，靠 rule IDs、scope/status/authority 過濾 |
| 圖片/文本跨模態檢索 | 高；沒有 image/text vectors 便不能做語義跨模態 | 只可用 alt text/OCR 的詞法搜尋 |
| 其他 Agent 使用知識 | 低；MCP 契約不變，只把 `retrieval_mode` 設成 lexical | 可以 |

因此，它不是「知識庫不可用」，而是從「能理解改寫和語義相似」降級為「精確詞法 + 結構化關係 + 來源證據」的知識庫。對第一批 Android 規則而言，這通常足以啟動；對跨語言問答、重複 Entity 發現和大規模維護，差距會逐漸變大。

### 23.2 三種 Profile

| Profile | SQLite | LanceDB | 建議用途 |
|---|---|---|---|
| `core-lexical` | FTS5 + metadata/relations/provenance | 不啟動、不進 MCP query path | **沒有 embedding 時的首選**；最低複雜度和完整 fallback |
| `lancedb-fts` | metadata/relations；可保留 SQLite FTS5 | LanceDB BM25 FTS index | 只有需要 LanceDB 的 phrase/boolean/fuzzy/array FTS 或希望預演其 hybrid API 時使用；不要同時查兩套 FTS |
| `hybrid` | 控制面 + FTS5 | vector/ANN + 可選 LanceDB FTS | 有 pinned local embedding 時的正式 profile |

LanceDB 官方文檔確認它可以對 string column 建 BM25 Full-Text Search index，支援 phrase、boolean、boosting、fuzzy 等查詢；新增資料後還需要 `optimize()` 才能把 FTS index 的 unindexed tail 合併。這代表「沒有 embedding 時 LanceDB 仍可做 FTS」，但對本項目來說，SQLite FTS5 已經提供足夠的單機詞法基線，因此不應為了「已安裝」而多維護第二套 FTS。

### 23.3 沒有 embedding 時 LanceDB 還有什麼作用

可以保留，但要把作用限制在以下其中一項，否則建議暫時不讓它進 runtime：

1. **Future vector schema：** 先定義 `chunk_id/content_sha256/modality` 等欄位，未來有模型時新增 vector column；目前不返回空向量或假分數。
2. **受控 FTS POC：** 以小型資料集比較 LanceDB BM25 和 SQLite FTS5 的召回、fuzzy/phrase 能力與維護成本；只有實測有收益才切換。
3. **Derived columnar asset store：** 當圖片/大型資產需要和 chunk metadata 一起按需讀取時，評估 LanceDB 的 columnar/binary 儲存；Git/sidecar 仍是來源，不能讓它取代 provenance。
4. **未來 hybrid migration target：** 有 embedding 後可使用 LanceDB 原生 vector + FTS hybrid 和 RRF；在此之前只保留 adapter，不讓 Agent 依賴 LanceDB 專有 table。

不建議的做法是：沒有 embedding 卻同時查 SQLite FTS5 和 LanceDB FTS，再把兩個詞法分數硬融合。這增加兩個 analyzer、tokenizer、index optimize 和一致性問題，通常不會帶來與維護成本相稱的收益。

### 23.4 Lexical-only MCP 行為

沒有 embedding 時，`get_build_info` 應返回：

```text
retrieval_profile: core-lexical
retrieval_mode: lexical_only
semantic_available: false
embedding_provider: disabled
embedding_model: null
degraded: false
reason: embedding_model_not_configured
```

`degraded: false` 表示這是明確選擇的正式 profile，不是一次事故；若原本的 hybrid build 失敗才退回 lexical，則使用 `degraded: true` 和 `reason: semantic_index_unavailable`。兩者都要讓 Agent 在回答中知道，避免它假裝做了語義搜尋。

Lexical-only 查詢仍應採用：

- canonical ID、alias、雙語 keywords 和 Android API coordinates。
- SQLite FTS5 的 `title/heading/body/aliases` 欄位和受控 query expansion。
- relation/status/authority/scope 先過濾，再返回 source locator。
- exact identifier 優先；不要用不透明的 fuzzy threshold 自動合併 Entity。
- golden cases 分別標記「詞面可解」和「需要語義」兩類，量化缺 embedding 的真實影響。

### 23.5 恢復 Embedding 的切換流程

```text
core-lexical source commit
  → 選定並 pin embedding model/provider
  → 對相同 stable chunk IR 建立新 LanceDB vector build
  → check_index_consistency.py
  → lexical vs hybrid golden/成本 benchmark
  → approved hybrid default
```

恢復語義層不需要修改 canonical Markdown、rule IDs 或 MCP 工具契約；只需新增 embedding model、LanceDB build 和 `retrieval_profile`。這正是把 LanceDB 放在 derived semantic plane、而不是 source layer 的價值。

## 24. Graph 方案對照與可吸收設計

### 24.1 先釐清「我們是否已經在用 Graph」

是，但目前是**邏輯圖（logical graph）**，不是專用圖資料庫。`entity`、`rule`、`process`、`source` 和 `asset` 是節點，`relation.schema.json` 定義的 typed relation 是邊；SQLite `relations` 表是這個圖的可重建投影。這種做法和 Neo4j/Kuzu 的差別在於儲存與查詢引擎，而不是資料是否具有圖結構。

因此不要把「使用 Graph」和「部署 graph database」混為一談：本方案可以先具備圖模型和圖檢索，等實際 benchmark 證明需要更複雜的圖運算後，才替換 storage adapter。

### 24.2 網上代表方案的核查與比較

| 方案 | 如何建圖 | 如何查詢 | 主要優勢 | 在目前約束下的問題 |
|---|---|---|---|---|
| Microsoft GraphRAG（standard） | TextUnit → LLM entity/relation/claim extraction → Leiden hierarchy → LLM community reports；另建 embedding | Global（community reports map-reduce）、Local（entity neighborhood）、DRIFT、Basic vector search | 對跨文件、多跳和整體主題問題有較強的 context 組織；有社群層級摘要 | 索引與查詢都圍繞 LLM/embedding；需要模型服務、成本和大量生成結果；Parquet/vector artifacts 也增加運維面 |
| Microsoft GraphRAG（fast method） | 用 NLTK/spaCy/共現替代部分 LLM extraction；community report 仍需 LLM | 仍使用 GraphRAG query modes | 比 standard 便宜、較快，可作探索性 POC | 官方明確指出圖較 noisy；不是無 LLM pipeline，且預設 NLP model 對中文/中英混合需另測 |
| Neo4j GraphRAG pattern | Property graph 節點/帶屬性關係；可由 vector/full-text 找 starting points | Cypher/GQL pattern、neighbor/path traversal、篩選與排序節點/關係/屬性 | 多跳、路徑可解釋、可把結構化資料和文字 chunk 一起查；適合複雜圖分析 | 通常要 Neo4j instance、driver、schema/index/權限與備份；引入常駐服務和新的運維面；vendor 的品質宣稱仍需自行 benchmark |
| LlamaIndex `PropertyGraphIndex` | `kg_extractors`；預設包含 LLM path extractor，也可用已有 node relationships 的 `ImplicitPathExtractor` | vector/synonym/Cypher/template/custom 多種 sub-retriever | 對 property graph 建構與查詢有成熟 orchestration；證明已有關係時可不需 LLM/embedding | 以 framework 封裝許多策略，依賴面較大；`TextToCypher`/LLM synonym 等路徑不符合目前無 API key 基線 |
| 本方案 Graph-lite | Git Markdown/YAML 中人工核准 entity/relation；deterministic compiler 投影 SQLite nodes/relations | FTS5 找 seed → recursive CTE/BFS 限深遍歷 → SQLite 權威/範圍/狀態過濾 → MCP 返回 path + provenance | 最低複雜度、可 diff/review/rollback、無 API key、可離線；和現有 schema/SSOT/SQLite 基線一致 | 不會自動發現新關係；跨語言改寫和全庫 global synthesis 較弱；關係品質靠 SME/審批和 deterministic checks |

官方資料的關鍵差異是：GraphRAG 的「Graph」通常還包括**建圖 pipeline、社群摘要、向量索引和查詢時 prompt 組裝**；我們要吸收的是檢索與治理中最有價值的子集，不複製需要模型服務的整條 pipeline。

### 24.3 哪一種在本項目比較有優勢

沒有絕對答案，取決於問題類型和限制：

| 問題/條件 | GraphRAG/專用圖資料庫較有優勢 | Graph-lite/SQLite 較有優勢 |
|---|---|---|
| 查詢需要跨多份文件連接 3 層以上關係 | 是；圖遍歷和 community context 可減少孤立 chunk | 可做 1–2 層受控 traversal；更深層需 benchmark 後才放寬 |
| 需要「整個知識庫的主題/趨勢/社群摘要」 | 是；Microsoft Global Search 直接針對 community reports | 只能先提供 deterministic counts、hub/orphan report 或人工維護 domain summary |
| 需要精確 API、規則、版本和來源定位 | 專用圖可做，但仍須額外 provenance 設計 | 是；SQLite FTS5 + typed relations + source locator 直接符合需求 |
| 無法使用 embedding、無 LLM API key | 標準 GraphRAG 不適合；FastGraphRAG 仍有 LLM report 步驟 | 是；可完整運行 lexical + explicit graph |
| 個人/內部 repo、希望 clone 後重建 | 需處理 DB 初始化、schema migration、索引/服務狀態 | 是；Git source + Python build + 單文件 SQLite，刪除 cache 可重建 |
| 團隊已有 Neo4j 平台和專職運維 | 可能值得直接採用 Neo4j adapter | 仍可先做 Graph-lite，再以同一 edge contract 對接 |

對目前的 VS Code GitHub Copilot Chat/Agent 場景，**Graph-lite 是整體優勢最高的第一版**：它保留 GraphRAG 最重要的多跳和可解釋能力，同時不引入 LLM API、常駐服務或第二份 canonical truth。專用 GraphRAG/Neo4j 的優勢應作為後續 POC 的比較基線，而不是現在的硬依賴。

### 24.4 建議吸收的 Graph 優勢（不增加核心複雜度）

#### A. 顯式節點/邊和證據

將現有 `relation.schema.json` 強化為可審查的 edge contract：

```yaml
id: rel.android.stateflow.ui-collection
kind: relation
source: entity.android.stateflow
predicate: UI_COLLECTS_WITH
target: entity.android.repeat-on-lifecycle
scope:
  platform: android
  min_version: "2.4.0"
evidence:
  - source_id: source.android.stateflow-sharedflow
    locator: "Collect flows safely / repeatOnLifecycle"
status: active
authority: official
review_by: 2027-01-01
```

規則：

- `source`、`target` 必須是已存在的 canonical ID；未知 ID、空 predicate、非法 self-loop 直接 fail。
- 每條 active edge 必須有 evidence、scope、authority 和 review date；沒有證據的候選只能進 `proposals/`。
- edge 的 canonical 定義留在 Git；SQLite 只保存 `edge_id`、索引欄位和 provenance 投影，LanceDB 不得成為新的關係真相。

#### B. 以 lexical seed 做 bounded graph retrieval

MCP 的 `list_related` 和 `search_knowledge` 可採用同一個 retrieval facade：

```text
query
  → SQLite FTS5 exact/alias seed
  → optional future LanceDB vector seed
  → recursive CTE / bounded BFS (max_depth=2, max_nodes=50)
  → status/authority/scope/review_by filter
  → path dedup + deterministic ranking
  → return nodes, edges, source locators, retrieval_mode
```

SQLite 官方文件明確支持 recursive CTE 遍歷 tree/graph，因此不需要為兩層鄰居查詢新增 graph database。安全上固定 `max_depth`、`max_nodes`、`max_edges` 和 timeout；MCP 不接受任意 Cypher/SQL。若未來向量可用，它只改變 seed 來源，不改變後續 edge traversal 或治理規則。

#### C. 把 path explainability 暴露給 Copilot

`search_knowledge` 的結構化輸出可增加：

```json
{
  "retrieval_mode": "lexical_graph",
  "seed_ids": ["entity.android.stateflow"],
  "paths": [
    {
      "nodes": ["entity.android.stateflow", "entity.android.repeat-on-lifecycle", "rule.android.flow.ui-collection"],
      "edges": ["rel.android.stateflow.ui-collection", "rel.rule.requires-lifecycle-collection"],
      "depth": 2,
      "evidence": ["source.android.stateflow-sharedflow"]
    }
  ],
  "degraded": false
}
```

Copilot 仍負責自然語言 synthesis，但可以說明「答案由哪條 path 和哪個 source 支持」。如果只返回一堆相似 chunk，模型較難知道關係是事實、候選還是偶然共現。

#### D. 用圖品質報告取代自動圖重構

`inspect_relations.py` 可以加入 deterministic graph health metrics：孤立節點、orphan edges、每種 predicate 的數量、入度/出度最高節點、跨 domain 邊、未被任何 golden case 使用的 active edge，以及 depth-2 查詢的節點/邊數上限。這些輸出是維護報告，不是自動搬目錄、合併 entity 或改寫 rule。

Leiden 或其他 community detection 只在離線 audit profile 執行，輸出「可能的模組邊界」報告；不得直接提交 canonical 變更。若將來需要全庫主題摘要，先由 SME 維護 `knowledge/decisions/` 或 domain summary，再考慮有授權的模型離線產生 candidate。

### 24.5 對現有架構的具體調整

| 位置 | 原有設計 | Graph-lite 調整 | 仍然不做的事 |
|---|---|---|---|
| Canonical `knowledge/` | entity/rule 有顯式 relations | 保持；補齊 edge ID、predicate、evidence、scope、review 狀態 | 不新增第二份 `relations.yml` 真相 |
| Schema | `relation.schema.json` 欄位級設計 | 增加 endpoint existence、predicate taxonomy、edge evidence 和 depth/query constraints 的驗證 fixture | 不讓模型輸出直接寫 active edge |
| SQLite | `relations` 表作控制面資料 | 增加 `nodes`/`edges`（或將現有 relations 視為 edges）投影、索引和 recursive CTE helper | 不把 SQLite 宣稱為完整 graph DB；不暴露任意 SQL |
| Retrieval | FTS5；未來 semantic/hybrid | `lexical_graph` 作正式 profile；vector 只作未來 seed；統一 path/provenance output | 不同時維護 SQLite FTS、LanceDB FTS 和 graph score 的不透明融合 |
| MCP | `search_knowledge`、`list_related` | `list_related` 支持 depth 0–2、node/edge/path 上限；search 返回 seed/path/evidence | 不暴露 Cypher、Kuzu query 或寫入工具 |
| CI/GitLab | deterministic schema/link/build checks | 加入 orphan edge、unknown endpoint、cycle/self-loop、bounded traversal 和 path golden cases | 不加入 GraphRAG LLM indexing、community report、LLM judge |
| Optional research | Leiden/圖資料庫列為後續 | 可建立 read-only export/POC，比較 Graph-lite 與 Neo4j/GraphRAG 的 recall、p95、RAM、維護成本 | 不因網上方案流行就把常駐 graph service 變成基線 |

### 24.6 更新後的實施路線

1. **Schema slice：** 先完成 `relation.schema.json`、predicate taxonomy、valid/invalid fixtures；確定所有 edge 都能追溯到 source。
2. **Compiler slice：** 把 canonical entities/rules/processes/sources 編譯成 SQLite `nodes`、`relations/edges` 和 FTS5；輸出 orphan/unknown endpoint 報告。
3. **Retrieval slice：** 實作 `list_related(depth<=2)` 和 `search_knowledge` 的 lexical seed → recursive traversal → provenance response；加入 10–20 個 multi-hop golden cases。
4. **Copilot slice：** 在 Custom Agent 指示中要求引用 path、edge ID 和 source；沒有 path/evidence 時明說無足夠證據，仍只產生 proposal。
5. **Benchmark slice：** 量測 lexical-only 與 lexical-graph 的 source-hit、multi-hop recall、path precision、p95 latency、節點/邊輸出大小；若沒有實質收益，保留 FTS-only。
6. **Future POC（按需）：** 有 embedding 政策批准後，讓 LanceDB vector search 提供 seed，沿用同一 Graph-lite traversal；只有在 depth、global query 或 graph analytics 明確成為瓶頸時，才比較 Neo4j/其他 graph store。

### 24.7 最終判斷

GraphRAG 不是 SQLite/FTS5 的直接替代品，而是一整套「建圖 + 社群摘要 + 向量/圖檢索 + LLM 生成」系統。對本項目的限制和目標，最佳折衷是：**Git/Markdown/YAML 作 SSOT；SQLite 同時保存 FTS5 和顯式關係投影；以 bounded recursive traversal 實現 Graph-lite；LanceDB 只保留未來 vector seed adapter；Copilot 在 VS Code 內作回答和 proposal 助手；GitLab CI 完全 deterministic。**

這樣既吸收 Graph 方案的多跳、可解釋和結構化過濾優勢，又維持 clone/build 可重建、無 API key、低運維和可審查。日後若資料量或問題型態真的需要 GraphRAG 的 global/community 能力，可以在不修改 canonical relation contract 和 MCP output contract 的前提下新增 adapter/離線 POC。

### 24.8 本節網上核查來源

以下是本節比較所依據的官方文檔或項目資料；GraphRAG/Neo4j 的品質改善描述仍應以自己的 golden cases 和成本 benchmark 驗證：

- [Microsoft GraphRAG overview](https://microsoft.github.io/graphrag/)：standard/fast indexing 概念、global/local/DRIFT/basic query。
- [Microsoft GraphRAG indexing overview](https://microsoft.github.io/graphrag/index/overview/)：TextUnit、entity/relation/claim、community、summary、embedding 和 Parquet/vector output。
- [Microsoft GraphRAG indexing methods](https://microsoft.github.io/graphrag/index/methods/)：standard 與 FastGraphRAG 的 LLM/NLP 取捨和 noise/cost 說明。
- [Microsoft GraphRAG model selection](https://microsoft.github.io/graphrag/config/models/)：LLM/embedding、LiteLLM、structured output、API key 和 custom protocol 邊界。
- [Neo4j What is GraphRAG?](https://neo4j.com/blog/genai/what-is-graphrag/)：vector/full-text seed、relationship/path traversal、filter/rank 和 explainability pattern。
- [LlamaIndex PropertyGraphIndex guide](https://developers.llamaindex.ai/python/framework/module_guides/indexing/lpg_index_guide/)：LLM path extractor、無 LLM/embedding 的 `ImplicitPathExtractor`、多種 sub-retriever。
- [SQLite recursive CTE](https://www.sqlite.org/lang_with.html)：使用 recursive common table expression 遍歷 tree/graph。

## 25. Entity lifecycle：新增、更新、刪除、合併與拆分

### 25.1 先說結論：Agent 不應直接「改 Entity」，而應提出可驗證的 lifecycle proposal

新知識進入時，Agent 可以協助找出既有 Entity、提出新增或變更草稿、列出可能的 duplicate pair、整理關係重寫清單；但它不應直接把 active Entity 改掉，也不應把相似度當成合併證據。正式流程固定為：

```text
Copilot/Agent research
  → proposal（operation + target IDs + base hashes + evidence）
  → SME/CODEOWNER adjudication
  → deterministic apply script
  → schema/link/lifecycle validation
  → SQLite/LanceDB rebuild
  → Git commit/MR + immutable build manifest
```

這個邊界比「給 Agent 一個 merge_entity 工具」更安全：Agent 的自然語言判斷只產生候選，真正改寫 ID、關係和索引由可測試的 Python 程式執行。

### 25.2 網上成熟做法可以借鑑什麼

Wikibase 官方的 merge 指引記錄了兩個很有用的 lifecycle pattern：合併時保留一個 survivor ID，並在 losing ID 之後建立 redirect；刪除則是有權限的操作，要提供理由/上下文並留下 deletion log。W3C PROV-O 則提供 `wasRevisionOf`、`wasDerivedFrom`、`hadPrimarySource`、`invalidatedAtTime` 等 provenance 概念。

我們不需要引入 Wikibase 或 RDF runtime，但應吸收其語義：

1. **合併不等於消失：** losing ID 保留為 `merged` tombstone/redirect，舊鏈接仍可解析。
2. **刪除不是靜默操作：** 預設做可追溯的 logical retire；真的 purge 必須有特殊政策、理由、批准和獨立 audit。
3. **每次變更都要有版本上下文：** 記錄 base commit/hash、proposal、actor、reviewer、時間、來源和 apply commit。
4. **衍生索引不參與裁決：** SQLite/LanceDB 只從批准後的 canonical state 重建；不能在索引中「順便」合併 Entity。

### 25.3 Lifecycle 狀態與操作語義

建議在 `taxonomy.schema.json` 受控枚舉中加入或確認以下狀態；實際命名可以按現有 schema 統一，但語義不能混淆：

| 狀態/操作 | Canonical 語義 | 查詢行為 | 是否需要 SME/CODEOWNER |
|---|---|---|---:|
| `draft` / `candidate` | 尚未批准的新 Entity 或候選變更 | 不進 active search；只可在 proposal/review 工具看到 | 是 |
| `active` | 可被 Agent 引用的正式 Entity | 正常返回，必須帶 provenance | 是（首次批准） |
| `update` | 修改名稱、定義、alias、scope、版本或關係；ID 通常不變 | 新 build 使用新版本；舊 commit 可回滾 | 是 |
| `retired`（logical delete） | 不再適用、重複但未合併、或業務停止使用 | active search 排除；`get_entity(old_id)` 返回 tombstone/reason | 是 |
| `merged` | losing Entity 已併入 survivor | 舊 ID 返回 `redirect_to`/`merged_into`；查詢可解析到 survivor | 是，且要審核衝突 |
| `split` | 原 Entity 被證明包含兩個或以上不同概念 | 原 ID 返回 `split_into`；新 IDs 分別 active | 是，通常高風險 |
| `purged` | 因法律/隱私等政策移除內容 | 不返回正文；只保留政策允許的最小 audit/tombstone | 特別批准 |
| `restore` | 從 retired/merged/split 或 Git 歷史恢復 | 由新 approved change 恢復，不直接覆蓋歷史 | 是 |

`delete` 不應作為日常 Agent 操作名；在 proposal 層使用 `retire`，把 `purge` 限定為受控人工流程。這可以避免「使用者說刪除」被誤解為不可回復的物理刪除。

### 25.4 Agent 如何判斷新增、更新、合併或刪除

| 發現情況 | Agent 建議的 operation | 必須附帶的判斷依據 | 預設決定 |
|---|---|---|---|
| 沒有相同 ID/alias，且有可靠來源 | `create` | canonical name、definition、scope、source locator、review date | 建立 `candidate`；批准後變 `active` |
| 命中既有 canonical ID | `update` | base content hash、差異 patch、受影響 relations、來源版本 | 不更換 ID；先做 optimistic concurrency check |
| 只在詞面上相似 | `candidate-duplicate` | aliases、definition、scope、authority、版本、共同/不同 relations | 不能自動 merge；產生待 adjudication pair |
| 兩個 Entity 定義和 scope 相同，證據支持同一概念 | `merge` | survivor 選擇、aliases 合併、incoming/outgoing edge rewrite、衝突清單 | SME 明確批准後才 apply |
| 一個 Entity 同時描述不同平台/版本/責任邊界 | `split` | 新 IDs、欄位分配、relation mapping、來源支持 | 阻止 merge；由 SME 分拆 |
| Entity 已過時但歷史引用仍有價值 | `retire` | reason、effective date、replacement/superseded ID | 保留 tombstone；active index 排除 |
| Entity 是錯誤或涉及政策要求移除的資料 | `purge` | 明確範圍、刪除 reason、批准人和刪除證據 | 不由普通 Agent 執行；同一 MR 受控處理 |

匹配優先級應是 `canonical ID → exact alias → 受控 identifier/coordinate → lexical candidate → future embedding candidate`。最後兩種只提供候選，不能直接觸發 merge。即使日後有 embedding，相似度也不能取代定義、scope、authority 和 evidence 比較。

### 25.5 Merge 的安全實作

一次 merge proposal 至少要包含：

```yaml
id: proposal.merge.stateflow-flow
kind: proposal
operation: merge
target_ids:
  - entity.android.state-flow
  - entity.android.kotlin-stateflow
survivor_id: entity.android.state-flow
base:
  source_commit: "<git-sha>"
  target_hashes:
    entity.android.state-flow: "<sha256>"
    entity.android.kotlin-stateflow: "<sha256>"
changes:
  aliases_to_add:
    - Kotlin StateFlow
  incoming_relations_to_rewrite: [rel.example.1, rel.example.2]
  outgoing_relations_to_review: [rel.example.3]
  conflicts: []
evidence:
  - source_id: source.android.stateflow-sharedflow
    locator: "StateFlow and SharedFlow"
risk: medium
requested_by: "copilot-session-or-engineer"
review_status: pending
```

deterministic `apply_proposal.py` 只在 `review_status: approved` 且 base hashes 沒有改變時執行：

1. 選定 survivor；將 losing Entity 設為 `status: merged`，寫入 `merged_into: survivor_id`，保留舊 aliases 和 provenance。
2. 將 incoming edges 指向 survivor；outgoing edges 逐條檢查，重複邊去重，語義不確定的邊建立 conflict 而不是猜測。
3. 把 losing ID 登記為 redirect；`get_entity(losing_id)` 返回 redirect 和 merge event，`search_knowledge` 可以標示 resolved ID。
4. 合併 aliases 前檢查 alias collision；一個 alias 仍對應多個不同 scope 時不得收斂成單一 alias。
5. 執行 schema、unknown endpoint、orphan edge、scope/authority、provenance 和 golden lifecycle tests，通過後才 rebuild indexes。

任何 scope、版本、authority 或定義不同的 pair 都要停在 `conflict`/`candidate-duplicate`，不能因相似度閾值或 Copilot 判斷直接合併。

### 25.6 Update、retire、purge、split 和 restore

**Update：** proposal 必須帶 `base_hash`，避免兩個 Agent 以舊內容互相覆蓋。只提交 field-level patch 和受影響 relation IDs；compiler 在 staging build 重新計算 content hash。若變更其實是「同名但不同概念」，改走 split，而不是覆蓋原 Entity。

**Retire（預設刪除）：** 保留原文件，但將 `status` 改為 `retired`，記錄 `retired_at`、`reason`、`replacement_id`（若有）、`approved_by` 和來源。active SQLite view 排除它，歷史 build 和 `get_entity` 仍可說明它為何退出。這是可回滾、可審查的 logical delete。

**Purge（特殊刪除）：** 僅在法律、隱私、版權或安全政策要求時使用。先建立獨立批准記錄，明確定義要移除的正文、附件、索引、cache、backup 和 Git history 範圍；若政策允許，留下不含敏感內容的最小 tombstone。完成後全量 rebuild 並掃描 orphan references。普通 Curator Agent 不得執行 purge。

**Split：** 新建兩個或以上穩定 IDs，逐欄位和逐 relation 指定去向；原 Entity 設為 `split` 並保存 `split_into` mapping。若無法判斷某條 relation 去向，保留為 conflict，不要把它隨機複製到所有新 Entity。

**Restore：** 優先使用 Git revert 或新的 reverse proposal，不直接修改歷史 commit。若恢復 merged Entity，需先檢查 survivor 期間是否已新增關係，必要時產生新的 conflict；restore 也必須重新 build 和對帳。

### 25.7 Schema 與 SQLite 投影需要增加什麼

正式 schema 尚未落地時，先把以下欄位加入 `entity.schema.json`、`proposal.schema.json`、`conflict.schema.json` 和 `taxonomy.schema.json` 的實作 backlog：

| 契約 | 建議欄位 | 用途 |
|---|---|---|
| Entity lifecycle | `status`、`merged_into`、`redirect_to`、`split_into`、`replacement_id`、`retired_at`、`retire_reason` | 表達 logical delete、merge、split、replacement，而不是物理刪除 |
| Proposal concurrency | `operation`、`target_ids`、`base.source_commit`、`base.target_hashes`、`proposed_patch`、`requested_by`、`review_status` | 防止 stale Agent 覆蓋新版本；讓 apply script 可重跑/拒絕 |
| Merge mapping | `survivor_id`、`loser_ids`、`alias_mapping`、`relation_rewrite`、`unresolved_conflicts` | 讓 merge 是明確 mapping，不是模糊指令 |
| Lifecycle event | `event_id`、`event_type`、`occurred_at`、`actor`、`approver`、`proposal_id`、`git_commit`、`source_ids` | 對應 revision/derivation/invalidation provenance |
| Conflict | `claim_ids`、`entity_ids`、`difference_type`、`owner`、`due_date`、`resolution`、`status` | 定義何時必須停下來人工 adjudicate |

SQLite 控制面可以增加或投影以下資料：

```sql
CREATE TABLE entity_redirects (
  old_id TEXT PRIMARY KEY,
  new_id TEXT,
  reason TEXT NOT NULL,
  event_id TEXT NOT NULL,
  effective_at TEXT NOT NULL
);

CREATE TABLE lifecycle_events (
  event_id TEXT PRIMARY KEY,
  event_type TEXT NOT NULL,
  target_ids_json TEXT NOT NULL,
  proposal_id TEXT NOT NULL,
  git_commit TEXT NOT NULL,
  actor TEXT NOT NULL,
  approver TEXT,
  occurred_at TEXT NOT NULL
);
```

對 `merged` Entity，`entity_redirects` 讓舊 ID 可解析；對 `retired` Entity，保留 lifecycle event 但不建立 redirect，除非有明確 replacement。所有表都是 build artifact，canonical lifecycle state 仍在 Git 文件和 Git history。

### 25.8 Agent、MCP 和 GitLab 的責任邊界

| 元件 | 可以做 | 不可以做 |
|---|---|---|
| Advisor Agent | 搜尋候選 Entity、顯示 relations/provenance、指出可能 duplicate | 寫 canonical、執行 merge/purge |
| Curator Agent | 在 branch 的 `knowledge/proposals/` 建立或修改 proposal、產生 field patch 和 relation mapping | 把 proposal 標為 approved、直接更新 active Entity |
| Reviewer Agent | 檢查 evidence、scope、authority、衝突、merge/split mapping；輸出 review checklist | 以模型自動判定高風險 merge 已批准 |
| 人類 SME/CODEOWNER | 批准或拒絕 create/update/retire/merge/split；處理 conflict | 跳過 schema/CI 或在索引中手改結果 |
| `apply_proposal.py` | 讀取 approved proposal，做 deterministic patch、redirect、relation rewrite 和 validation | 呼叫 LLM、猜測 entity identity、繞過 base hash |
| MCP gateway | 只讀 `search/get/list_related/get_build_info`；返回 redirect/path/provenance | 暴露任意 graph query、提供無審批 write/delete tool |
| GitLab CI | schema、lifecycle、orphan、golden、build、consistency checks | 呼叫 Copilot/LLM API、批量自動合併 Entity |

Custom Agent 的 prompt 不是安全邊界；真正的保護要靠 branch protection、CODEOWNERS、proposal-only path allow-list、CI 必須通過和 apply script 的 approval/hash checks。即使 Copilot 在 VS Code 內能編輯文件，也不能因此把它視為已獲授權的 canonical writer。

### 25.9 Lifecycle 驗證與回滾測試

最少加入以下 deterministic fixtures/golden cases：

1. `create_new_entity`：新 ID、來源和 required fields 齊全後才可 active。
2. `update_stale_base`：base hash 過期時 apply 失敗，不覆蓋現有文件。
3. `merge_same_scope`：survivor、aliases、incoming edges、redirect 和 provenance 正確。
4. `merge_scope_conflict`：同名但 Android 版本/scope 不同時保持 conflict，不合併。
5. `retire_with_replacement`：active search 排除 retired，舊 ID 返回 reason/replacement。
6. `purge_policy_gate`：缺少明確 reason、刪除範圍或 required approvers 時拒絕 purge。
7. `split_relation_mapping`：沒有指定去向的 relation 使 proposal fail 或進 conflict queue。
8. `restore_after_merge`：restore 不破壞 survivor 期間新增的關係，必要時要求人工解衝突。
9. `redirect_chain_limit`：A→B→C 的 redirect 需要壓縮或最多解析固定跳數，禁止無限循環。
10. `rebuild_after_lifecycle`：SQLite/LanceDB（若啟用）只包含批准後狀態，沒有 orphan 或 stale vector。

每個 lifecycle build manifest 都應記錄 `proposal_ids`、`lifecycle_event_ids`、canonical commit、schema/compiler version 和 artifact hashes。回滾時切換到上一個 immutable build 或 revert Git commit，而不是直接在 SQLite 裡刪行。

### 25.10 更新後的 roadmap

1. **Schema slice：** 先落地 lifecycle enum、redirect/merge/split 欄位、proposal base hash 和 conflict schema。
2. **Proposal slice：** 實作 deterministic proposal template/validator；Curator Agent 只寫 proposal path。
3. **Apply slice：** 實作 `apply_proposal.py`，支援 create/update/retire/merge/split/restore，先不支援普通 Agent purge。
4. **Graph-lite slice：** 對 approved lifecycle state 建立 nodes/edges；測試 redirect resolution、relation rewrite 和 bounded traversal。
5. **CI slice：** 加入 lifecycle fixtures、CODEOWNERS、stale base、orphan/redirect-cycle 和 rollback build checks；保持不需 LLM API。
6. **Future candidate slice：** 有 embedding 後才用 vector/lexical 相似找 duplicate candidates；仍必須經同一 proposal/adjudication/apply 流程。

### 25.11 本節網上核查來源

- [Wikibase/Merging](https://www.mediawiki.org/wiki/Wikibase/Merging)：merge survivor ID 和建立 redirect 的官方指引。
- [Wikibase/Creating and deleting data](https://www.mediawiki.org/wiki/Wikibase/Creating_and_deleting_data)：建立/刪除工具、刪除理由與 deletion log 的官方指引。
- [W3C PROV-O](https://www.w3.org/TR/prov-o/)：`wasRevisionOf`、`wasDerivedFrom`、`hadPrimarySource`、`invalidatedAtTime` 等 provenance 語義。

## 26. 人工批准如何透過 GitLab MR 落地

### 26.1 先回答問題：批准和 merge 是兩件事

在本方案中，**人工批准的正式載體是 GitLab Merge Request（MR）的 approval record**；MR 的 `merge` 是在批准、CI 和分支保護條件都滿足後，由有權限的 Maintainer 執行的發布動作。

```text
Agent/工程師建立 branch
  → proposal +（必要時）deterministic canonical diff
  → 開 GitLab MR
  → SME/CODEOWNER 在 GitLab review diff、留言、Approve
  → required approvals + CI passed + no unresolved thread
  → Maintainer merge 到 protected main
  → post-merge deterministic build / manifest
```

不要把以下三件事混為一談：

| 動作 | 意義 | 是否等於批准 |
|---|---|---:|
| Agent 產生 proposal | 候選變更；沒有權威性 | 否 |
| SME/CODEOWNER 點擊 GitLab `Approve` | 對 MR 中的變更作正式 review decision；可被撤回或因新 commit 失效，按 GitLab 設定重新批准 | 是，這是本方案的人工批准事件 |
| Maintainer 點擊 `Merge` | 將已批准且 CI 通過的 MR 寫入 protected `main` | 否，是發布/整合動作 |

GitLab 官方文件指出，required approvals 可阻止未達批准數的 MR merge；CODEOWNERS 可對指定路徑要求專家批准，但 required approval rules 和 Code Owner enforcement 受 GitLab tier/Protected Branch 設定影響。GitLab Free 的一般 approvals 可能只是 optional，不能假定它會自動阻止 merge；若需要硬門禁，應確認 Premium/Ultimate 或公司 GitLab Self-Managed 的等價配置。

### 26.2 推薦的低風險單一 MR 流程

對普通 `create`、低風險 `update` 和 `retire`，採用一個 MR 即可，但 MR 必須同時包含：

```text
knowledge/proposals/<proposal-id>.yml    # Agent/工程師的候選操作
knowledge/entities/<id>.md                # apply_proposal.py 產生的 canonical diff
knowledge/rules/<id>.md                   # 如受影響
```

操作步驟：

1. Curator Agent 只在 branch 的 `knowledge/proposals/` 建 proposal，填入 operation、target IDs、evidence、base hashes 和預期 relation 變更。
2. 工程師在同一 branch 執行 `apply_proposal.py --preview` 或受控 branch apply，產生實際 canonical diff；這不是批准，只是讓 reviewer 看見將要合入的內容。
3. GitLab MR 同時展示 proposal、canonical diff 和受影響的 relations/redirects；CI 重新 dry-run apply，確認提交的 canonical diff 與 proposal 產出一致。
4. SME/CODEOWNER review 實際 diff，確認 definition、scope、authority、來源、lifecycle 狀態和 relation mapping，然後在 GitLab 點擊 `Approve`。
5. 所有 required approvals、CI、security/path checks 和 unresolved-thread 條件滿足後，Maintainer 才 merge。
6. merge 後 pipeline 從 `main` 重建 SQLite/LanceDB（若啟用），寫入 build manifest；任何 cache/index 都不被 reviewer 當作批准對象。

這種方式的重點是：**批准的是 MR 中可見的 canonical diff，而不是一個模型文字建議。** MR branch 可以包含未批准的候選內容，但 protected `main` 不會接受它。

### 26.2.1 批准後不是回到 branch 再手動 pull：誰執行 rebuild？

這裡沒有「Approve 後由某人再 pull branch，然後才 rebuild」這個隱藏步驟。`pull` 只是一個本地開發者想把最新 `main` 帶回自己電腦時才會做的動作；它不是 GitLab 發布流程的一部分。GitLab Runner 會在 merge 後的 pipeline 自動 checkout 合入後的 `main` commit（例如 `CI_COMMIT_SHA`），再執行 deterministic build。

把兩個 pipeline 分開看，流程就不會中斷：

| Pipeline | 執行版本 | 用途 | 是否發布正式索引 |
|---|---|---|---:|
| MR pipeline | proposal branch／merged-result（按 GitLab 設定） | validate schema、dry-run apply、測試、預覽 build | 否；最多產生臨時 artifact 供 reviewer 檢查 |
| post-merge pipeline | 已合入 `main` 的 commit | 從 SSOT 重新編譯 SQLite、可選 LanceDB、manifest 和 consistency report | 是；把 CI artifact 發布給需要共享索引的消費者 |
| 本機查詢流程 | 使用者自己 clone/pull 後的工作樹 | `python scripts/build_indexes.py` 建立本地 cache，啟動本地 MCP | 只影響該使用者本機 |

推薦的實際時間線是：

```text
MR branch
  ├─ proposal + canonical diff
  ├─ MR pipeline：validate / preview / test
  └─ SME/CODEOWNER Approve
       ↓
Maintainer Merge → main commit M
       ↓
GitLab post-merge runner 自動 checkout M
       ↓
build_indexes.py → check_index_consistency.py
       ↓
SQLite/LanceDB snapshot + manifest 作為 CI artifact
```

因此，批准者不需要在批准後操作 branch，也不需要把索引檔手動搬回 repository。索引是 canonical Git 內容的衍生物：

- 若每個開發者都在本機執行 MCP，CI 可以只保存 build artifact；使用者在 clone 或 pull 後自行 rebuild 本地 `.cache/`。
- 若團隊有共享 MCP/服務，post-merge pipeline 應把 immutable snapshot 發布到受控 artifact/package storage，再由部署步驟切換 `current.json`；這仍然不是 reviewer 手動 pull。
- 不建議把 SQLite/LanceDB cache commit 回 `main`，否則每次索引重建都會製造巨大、難審查的二進制 diff，也會破壞「Git Markdown/YAML 是 SSOT」的邊界。

如果 post-merge build 失敗，MR 已經合入的 canonical commit 仍然是唯一真相；pipeline 應標記 release/index artifact 未發布並告警，修復後從同一個 commit 或其後續 commit 重跑 build，而不是由人工修改 SQLite 來補救。

### 26.3 所有變更統一使用單一 MR

為了維持低複雜度，`create`、`update`、`retire`、`merge`、`split`、一般 conflict 和 `purge` **全部只使用一個 MR**。在開 MR 前由 `apply_proposal.py --preview` 產生**完整、可重現、可審查的最終 canonical diff**；同一個 MR 同時審查「變更範圍」和「實際結果」：範圍由 allow-list/scope manifest 鎖定，結果由 canonical diff 鎖定。

```text
Draft MR（先不批准，直到最終 diff 生成）
  proposal + scope manifest + canonical diff
  → CI: deterministic preview / hash / allow-list / purge dry-run
  → SME/CODEOWNER（語義）+ KB Maintainer（結構）批准同一 MR
  → Maintainer Merge 到 protected main
  → post-merge runner rebuild indexes
  → 只有 purge：protected manual job 清理受控 artifacts/cache（不開第二個 MR）
```

單一 MR 必須具備以下控制：

| 控制 | 實作 | 失敗行為 |
|---|---|---|
| Scope lock | `scope_manifest.json` 列出 operation、target IDs、允許修改/刪除路徑、預期檔案數和 `scope_hash` | scope 與實際 diff 不一致，CI fail |
| Final diff lock | branch 先執行 `apply_proposal.py --preview --verify-diff`，把 canonical diff 一同放入 MR | 未生成完整 diff 不可請求批准 |
| Reproducibility | CI 在乾淨 checkout 重新 apply，驗證輸出 hash 與 branch diff 相同 | hash 不同或 `base_hash` 過期，CI fail 並要求更新 MR |
| Review separation | 同一 MR 使用 required approval rules：一般 conflict 由 SME + KB Maintainer 批准；purge 由 Policy Owner + KB Maintainer 批准（Policy Owner 可同時是 CODEOWNER） | 缺任何 required approval，不可 merge |
| Destructive allow-list | `purge_manifest` 明確列出正文、附件、索引、cache、backup 範圍和刪除 reason；執行前只允許該清單 | 超出清單立即停止，不做部分猜測式刪除 |
| Post-merge gate | merge 後先 rebuild、orphan scan、manifest/consistency check；purge 清理 job 使用 protected environment + `when: manual` | artifact 不發布；保留 canonical commit，修復後重跑 |

對 `purge`，一個 MR 仍可保留雙人控制，但「雙人」是**同一 MR 的兩個 required approvals**，不是兩個 MR。若需要在實際清理前再按一次按鈕，使用 GitLab protected manual job；這是執行閘門，不是重新審查另一個 canonical diff。除了這個執行閘門，不另設第二個 MR。

如果 GitLab tier 不支援 required approval rules，仍可用一個 MR 記錄決策，但必須由 protected branch、Maintainer checklist 和受保護的手動 job 補足控制；不能宣稱平台已硬性阻擋未批准 merge。

### 26.4 GitLab 專案設定建議

```text
Protected branch: main
  - no direct push for normal users/agents
  - merge only for Maintainer/release role

CODEOWNERS
  /knowledge/             @sme-knowledge-owners
  /knowledge/entities/   @sme-entity-owners
  /knowledge/rules/      @sme-rule-owners
  /schemas/              @kb-maintainers
  /.github/              @kb-maintainers
  /.vscode/              @kb-maintainers

Approval rules
  - Entity/rule change: at least 1 matching SME/CODEOWNER
  - merge/split/conflict: 1 SME + 1 KB maintainer
  - purge/security-sensitive asset: designated policy owner + maintainer

Merge checks
  - required approvals satisfied
  - MR pipeline passed
  - no unresolved threads
  - no merge conflict
  - stale-base/lifecycle validation passed
```

GitLab 官方文件也提醒，CODEOWNERS 要與 protected branch 和 required Code Owner approval 一起設定；如果允許某個 automation account 直接 push/merge `main`，它可以繞過 MR/Code Owner 流程。因此 Agent、CI job token 和 release bot 不應被授予普通的 direct push/merge 權限。

### 26.5 `review_status`、audit 和 build manifest 如何同步

GitLab 的 approval record 是平台 audit；Git 中的 proposal 是可移植的 domain record，兩者各有責任：

| 記錄 | 儲存位置 | 內容 |
|---|---|---|
| Review decision | GitLab MR | MR IID、review comments、Approve/Revoke、approver、時間、pipeline status |
| Proposal state | `knowledge/proposals/*.yml` | `pending/approved/rejected/applied/superseded`、target IDs、base hashes、evidence、approval reference |
| Lifecycle event | canonical/event projection | event ID、operation、actor、approver、proposal ID、apply commit、source IDs |
| Published build | `.cache/builds/<build-id>/manifest.json` | source commit、proposal/event IDs、schema/compiler versions、SQLite/LanceDB hashes |

在沒有 GitLab API key 的前提下，不要求 CI 自動查詢 approver 名單。人工批准後，由 Maintainer 在同一 MR 的受控 commit 將 `review_status`、`approved_by` 和 MR IID 寫入 proposal；CI 只驗證欄位格式、commit/base hash 和 GitLab pipeline gate。GitLab 本身保留不可由模型偽造的 approval audit。

如果團隊希望完全避免手動同步，亦可不把 `approved_by` 當 canonical truth，而只把 `review_status: applied`、`git_commit` 和 `merge_request_iid` 寫入 post-merge event；批准者以 GitLab audit 為準。兩者擇一後固定，不要讓不同資料源互相覆蓋。

### 26.6 GitLab CI 的具體 gate（仍然不需 LLM API）

MR pipeline 只做 deterministic checks：

```yaml
stages: [validate, test, preview, build]

validate_lifecycle:
  stage: validate
  script:
    - .venv/bin/python scripts/validate_kb.py
    - .venv/bin/python scripts/validate_lifecycle.py

test:
  stage: test
  script:
    - .venv/bin/python -m pytest

preview_apply:
  stage: preview
  script:
    - .venv/bin/python scripts/apply_proposal.py --all --dry-run --verify-diff

build_lexical:
  stage: build
  script:
    - .venv/bin/python scripts/build_indexes.py --profile core-lexical
    - .venv/bin/python scripts/check_index_consistency.py --profile core-lexical

post_merge_publish:
  stage: build
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH && $CI_PIPELINE_SOURCE == "push"'
  script:
    # Runner 已在 merged main commit 上；不需要人工 pull
    - .venv/bin/python scripts/build_indexes.py --profile core-lexical --output .cache/builds/$CI_COMMIT_SHA
    - .venv/bin/python scripts/check_index_consistency.py --profile core-lexical --build .cache/builds/$CI_COMMIT_SHA
  artifacts:
    paths:
      - .cache/builds/$CI_COMMIT_SHA/
    expire_in: 30 days
```

CI 不需要呼叫 GitLab approvals API、Copilot、OpenAI/Anthropic API 或其他 LLM。平台層的 required approval 和 protected branch 先決定 MR 能否 merge；MR pipeline 只決定內容是否通過 deterministic validation，post-merge pipeline 則從已合入 commit 建立和發布衍生索引。若日後加入 GitLab API 讀取 approval status，也只能作額外報告，不能取代 protected branch/required rule 的平台設定。

### 26.7 Copilot 在這個流程中的實際位置

在 VS Code 內，Copilot 可以：

- 讀新來源並建立 `candidate` Entity proposal。
- 找出 exact ID、alias collision 和可能 duplicate pair。
- 產生 update patch、merge mapping、split mapping 和 conflict checklist。
- 協助撰寫 MR description 和 reviewer checklist。
- 根據 MCP 返回的 path/provenance 解釋為何提出變更。

Copilot 不可以：

- 點擊 GitLab `Approve`。
- 直接 merge MR 或繞過 protected branch。
- 把 `review_status` 改成 approved 來偽造人工批准。
- 在沒有 evidence 時自動 merge/delete Entity。
- 直接修改 SQLite/LanceDB 來掩蓋 canonical diff。

所以「人工批准」不是 Copilot 的另一個 prompt，而是 GitLab 身份、MR review、CODEOWNERS、protected branch 和 CI gate 共同形成的外部治理結果。

### 26.8 建議的日常操作範例

**普通新增：** Curator Agent 建 proposal → branch preview apply → 單一 MR → SME approve → Maintainer merge → build。

**疑似重複：** Advisor 只產生 duplicate candidate → SME 決定 merge/keep-separate → branch 產生完整 deterministic mapping → 同一 MR review/approve/merge，並檢查 redirect/relation mapping。

**過時規則：** Agent 建 `retire` proposal → reviewer 確認 replacement 和 effective date → 單一 MR 通過後 active search 排除，歷史仍可查。

**法律刪除：** Policy owner 建 purge proposal + `purge_manifest` + 明確 reason → 同一 MR 由 Policy Owner + KB Maintainer 按 required rules 批准 → Maintainer merge → post-merge rebuild/orphan scan → protected manual job 清理受控 artifacts/cache；普通 Agent 不參與批准或執行。

### 26.9 本節網上核查來源

- [GitLab merge request approvals](https://docs.gitlab.com/user/project/merge_requests/approvals/)：required/optional approvals、approval status、批准與 merge blockers。
- [GitLab Code Owners](https://docs.gitlab.com/user/project/codeowners/)：CODEOWNERS、protected branch 和 required Code Owner approval 的關係。
- [GitLab merge request pipelines](https://docs.gitlab.com/ci/pipelines/merge_request_pipelines/)：使用 `CI_PIPELINE_SOURCE == "merge_request_event"` 執行 MR deterministic pipeline。

## 27. LLM Wiki 對照：實際 Entity 操作與可借鑑設計

### 27.1 本節的「LLM Wiki」指哪些項目

網上有多個同名或相近項目，不能把它們當成一個產品：

| 項目 | 本次核對版本／定位 | 對本方案的參考價值 |
|---|---|---|
| [`nashsu/llm_wiki`](https://github.com/nashsu/llm_wiki) | commit `e8082119649e6a8e1cf85eaf289adcabfdf39d4e`，package version `0.6.11`；Tauri 桌面應用，LLM 直接 ingest、維護 wiki，附 graph、LanceDB、Review UI、HTTP API/MCP | 最接近大家口中的「LLM Wiki」；可觀察實際 ingest、source delete、duplicate merge 行為 |
| [Karpathy LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) | Raw sources → durable wiki pages → `index.md`/`log.md`/wikilinks；人類策展、LLM 維護的方法論 | 支持 Markdown wiki 作為持久化知識層，但沒有嚴格 Entity lifecycle |
| [`atomicstrata/llm-wiki-compiler`](https://github.com/atomicstrata/llm-wiki-compiler) | commit `cbd09c6c415f36b6001adf89a02aa805a5a5aba6`，package version `1.1.0`；Node/TypeScript compiler，提供 Configurable Lifecycle Profiles、typed relations、review candidates、trust gates | 最值得借鑑其 profile/schema、重驗證與 fail-closed 寫入；不是我們的 Python/no-API runtime 依賴 |

以下「LLM Wiki 的實際做法」主要指第一個 repository；第三個項目的 profile/lifecycle 會另行標示。這樣可以避免把 Karpathy 的抽象 pattern、桌面應用和 compiler 的功能混成一個不存在的規格。

### 27.2 `nashsu/llm_wiki` 實際怎樣處理新增、更新、刪除、合併

我按上述 commit 的 README 和 source code 交叉核對，得到以下結果：

| Entity／內容操作 | `nashsu/llm_wiki` 的實際行為 | 治理特性 | 與我們的差異 |
|---|---|---|---|
| Create | ingest 先以 LLM analysis，再以 LLM generation 產生 `wiki/.../*.md`；`sources[]` 寫入來源追溯。應用層直接把合法 FILE blocks 寫到工作樹 | 有路徑安全檢查和 source traceability，但不是 Git MR 寫入 | 我們把候選寫入 `knowledge/proposals/`，canonical 只由批准後 deterministic apply 產生 |
| Update／re-ingest | 同一路徑已存在時，`mergePageContent` 會 union `sources/tags/related`；正文不同則呼叫 LLM merge，鎖定 `type/title/created`，更新 `updated`；單一來源修正可替換舊正文 | 有 fallback、backup、body shrink sanity check | 我們要把 field-level patch、base hash、受影響 relation IDs 固定在 proposal，避免模型直接重寫正文 |
| Source delete | `source-lifecycle.ts` 先刪 raw source；掃描每頁 `sources[]`。仍有其他來源的頁面保留並移除被刪來源；沒有 survivor 的頁面進 cascade delete；同時清 index、wikilink、`related:`、向量和 source media | shared-source preservation 做得實用；外部 watcher 對 created/modified/deleted 重用同一 lifecycle | 我們保留 Git history 與 `retired`/provenance tombstone，刪除預設不是物理刪除；來源撤回應產生 MR |
| Entity/page direct delete | `wiki-page-delete.ts` 物理刪檔，之後移除 embedding，並掃描其他頁清理指向該 slug/title 的引用 | 有引用清理和防止 substring 誤刪的測試 | 我們對日常 Entity 使用 `retire`；`purge` 只在 policy/legal gate 下執行 |
| Duplicate candidate | `dedup.ts` 先從頁面 frontmatter/body 建 summaries；LLM 回傳 duplicate groups、reason、confidence。可保存「not duplicates」白名單 | 候選不是自動 merge；需使用者確認 | 我們即使未來有 embedding 也只把它當候選生成器，不能直接決定 merge |
| Merge | 使用者在 Maintenance UI 確認 group 和 canonical slug 後，`executeMerge` 呼叫 LLM 合併正文；應用層 deterministic union frontmatter、改寫 wikilinks/`related:`、備份到 `.llm-wiki/page-history/`，最後物理刪除 losing pages；queue 會 serialise、retry、可取消 | 有「人先確認」和 backup，適合個人應用 | 沒有 survivor/loser 的穩定 Entity ID、redirect/tombstone、relation mapping/conflict gate；這正是我們要補強之處 |
| Split | 在該 commit 的公開 lifecycle／dedup 路徑沒有 first-class split operation 或 `split_into` mapping | 通常只能人工另建頁面 | 我們明確要求新 IDs、逐欄位／逐 relation mapping，原 ID 進 `split` tombstone |
| Review | ingest 產生 `.llm-wiki/review.json` item；Review UI 可 resolve、dismiss、create page、delete page 或 research；review state 是本地應用資料 | 是 asynchronous human-in-the-loop，但不是 Git identity／MR approval | 我們將「批准 canonical diff」放在 GitLab MR/CODEOWNERS，平台 audit 不由模型偽造 |

這個結果回答了「它怎樣處理 Entity merge/delete」：它偏向**應用內、LLM 輔助、使用者點擊後即時改工作樹**。它有不少工程上的清理和備份，但 Entity identity semantics 比較弱；losing page 被刪掉後，舊連結靠當次 rewrite 消失，沒有可長期解析的 redirect。

### 27.3 為什麼不能直接照搬 `nashsu/llm_wiki`

`nashsu/llm_wiki` 的優勢是完整產品體驗，而不是符合本項目的部署約束：

- ingest、duplicate detection、正文 merge、image caption、Deep Research 和 optional vector search 都圍繞 LLM provider；README 的 vector search 也要求 OpenAI-compatible embeddings endpoint/API key。
- App 內的 review queue 解決「稍後由使用者點選」，但不等於 GitLab 的 branch protection、CODEOWNER identity、MR diff 和可回滾 release record。
- 對 source deletion，它的 source ownership 是實用的；對 typed Entity，它沒有我們需要的穩定 ID、scope/authority/lifecycle 和 relation rewrite contract。
- 物理刪除 losing page 對個人 wiki 很方便，對內部 SME 知識則可能破壞舊引用、歷史審計和跨版本查詢。

因此我們吸收其「增量 compilation、source traceability、shared-source preservation、deterministic reference cleanup、backup」；不吸收「LLM 直接寫 canonical、物理 merge delete、應用內 review 取代 repository approval、embedding endpoint 作硬依賴」。

### 27.4 `atomicstrata/llm-wiki-compiler` 最值得借鑑的部分

第三個項目把原本的 wiki convention 提升為可驗證 contract，這對我們比桌面 UI 更有參考價值：

| 借鑑點 | 它的做法 | 我們應採用的 Python 版本 |
|---|---|---|
| Profile as data | `.llmwiki/profile.json` 宣告 entity types、fields、relations、lifecycles、workflows、artifacts | `schemas/*.schema.json` + `configs/taxonomy.yml` 宣告同樣 contract；compiler 不把 Android 規則硬編碼成分支 |
| Lifecycle FSM | lifecycle field 有 initial/terminal/transitions；每次 typed write／transition 都重驗證 | `validate_lifecycle.py` 驗證 enum、合法 transition、required evidence；`apply_proposal.py` 只接受明確 operation |
| Review candidate | `compile --review` 寫 candidate；approve 時 re-read、hash、re-plan，再寫 live page；不在 approve 時重新呼叫 LLM | proposal MR 中保存 `base.source_commit`、target hashes、proposed patch；apply 前重新讀取並 fail-closed |
| External input staging | OKF import／connector 預設只進 review queue；trusted import 是明確例外 | 任何 Copilot 產生內容、外部匯入或 agent 草稿都只能進 `proposals/`；不可由 MCP write tool 直接寫 active |
| Typed relations | relation type、endpoint、attributes、evidence 有 profile contract | Git canonical relation + SQLite projection；unknown endpoint、非法 predicate、缺 evidence 在 CI 阻斷 |
| Deterministic rebuild | compile、lint、review、export、MCP 共用同一 profile/schema | `build_indexes.py` 從 approved Git source 重建，不把 SQLite/LanceDB 當第二真相 |

但它仍不是直接依賴：其 `package.json` 帶有 Anthropic/Claude Agent、OpenAI、MCP SDK 等 Node 依賴；我們目前刻意不把它的 LLM ingestion、provider adapters 或 TypeScript runtime 放進 GitLab pipeline。

### 27.5 Entity 操作在我們架構中的最終落地方式

把兩個項目的優點合在一起後，推薦的 state/change 邊界如下：

```text
新來源／新觀察
  → Copilot 在 VS Code 讀取 source + MCP evidence
  → proposal（create/update/candidate-duplicate/retire/merge/split/restore）
  → deterministic preview：apply_proposal.py --dry-run
  → GitLab MR 顯示 proposal + canonical diff + relation mapping
  → SME/CODEOWNER Approve（平台身份）
  → Maintainer Merge 到 protected main
  → post-merge runner 從 merged commit rebuild SQLite／可選 LanceDB
```

具體語義固定為：

1. **Create：** 新 Entity 先以 `candidate` 出現；需有 canonical ID、定義、scope、authority 和 evidence，批准後才進 `active`。
2. **Update：** ID 不變；proposal 帶 `base_hash` 和 field-level patch。base 過期即拒絕，不讓較舊 Agent 覆蓋較新內容。
3. **Candidate duplicate：** lexical（未來可加 vector）只產生候選 pair；Agent 必須列出 definition/scope/authority/relations/evidence 差異，不能以相似度直接 merge。
4. **Merge：** SME 選 survivor；apply script 合併 aliases、逐條重寫 incoming/outgoing relations、檢查 collision；loser 設 `status: merged`、`merged_into` 和 redirect/tombstone，保留 provenance。語義不確定的 edge 進 conflict，不猜。
5. **Retire：** 一般「刪除」只把 Entity 設 `retired`，記錄 reason、effective date、replacement；active search 排除，但舊 ID 可查 tombstone。
6. **Split：** 建立新 IDs 和 relation mapping；原 Entity 設 `split` 並保存 `split_into`；任何未分配 relation 都使 apply 失敗或進 conflict queue。
7. **Purge：** 只有法律、隱私、版權或安全政策才使用；指定 policy owner + Maintainer 雙人批准，處理正文、附件、索引、cache、backup 和必要的 Git history 範圍。
8. **Restore：** 以新的 approved proposal 或 Git revert 恢復；先檢查 survivor 之後新增的 relations，避免 restore 把新知識覆蓋。

這個設計比 `nashsu/llm_wiki` 多了一層 proposal/apply 和 lifecycle metadata，但仍比完整 GraphRAG／桌面服務低複雜度；它把最容易出錯的 Entity identity、relation rewrite 和 destructive delete 變成可測試的 deterministic code，而把 Copilot 保留在最擅長的 research/draft/synthesis 位置。

### 27.6 對本項目的取捨結論

| 判斷 | 結論 |
|---|---|
| 是否採用 LLM Wiki 的 wiki pattern？ | 是：raw/source evidence、Markdown pages、index/log、incremental compile、citations |
| 是否採用其桌面應用架構？ | 否：我們的入口是 VS Code GitHub Copilot Chat/Agent + read-only MCP，不另建 Tauri/HTTP desktop backend |
| 是否採用 LLM 自動 ingest／merge？ | 只作 VS Code 互動式 proposal；GitLab CI 不呼叫 LLM API，canonical write 必須經 MR |
| 是否採用其 source delete cleanup？ | 吸收 shared-source 判斷和 link/index cleanup；改成 proposal + logical retire，物理 purge 受控 |
| 是否採用其 graph／LanceDB？ | Graph-lite + SQLite relation projection 保留；LanceDB 是 future semantic plane，無 embedding 時不進預設 query path |
| 是否採用 atomicstrata 的 lifecycle profile？ | 吸收 profile/schema、FSM、review candidate revalidation、fail-closed trust gate；以 Python/JSON Schema 重做 |
| Entity merge 的最終權威在哪裡？ | Git canonical files + GitLab MR approval；不是 Copilot review item、SQLite、LanceDB 或模型 confidence |

### 27.7 本節核查來源（repository 與精確 source path）

以下連結均以本節記錄的 commit 固定版本；研究日期為 2026-09-03：

- [`nashsu/llm_wiki` README](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/README.md)：two-step ingest、source traceability、source watch、graph、Review UI、optional LanceDB/API。
- [`nashsu/llm_wiki` ingest merge path](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/ingest.ts)：generated FILE blocks 及 existing page 的 merge/write 行為。
- [`nashsu/llm_wiki` page merge](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/page-merge.ts)：array union、LLM body merge、locked frontmatter、fallback/backup。
- [`nashsu/llm_wiki` duplicate detector/merge](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/dedup.ts)：summary → LLM duplicate group → canonical body merge、reference rewrite、pagesToDelete。
- [`nashsu/llm_wiki` duplicate executor](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/dedup-runner.ts)：使用者確認後的 backup、write、rewrite、物理刪除。
- [`nashsu/llm_wiki` source lifecycle](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/source-lifecycle.ts)：source ownership、shared page 保留、cascade delete、log。
- [`nashsu/llm_wiki` wiki-page delete](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/wiki-page-delete.ts)：embedding/media/reference cleanup。
- [`nashsu/llm_wiki` review store](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/stores/review-store.ts)：本地 review item、resolved/dismiss state。
- [`nashsu/llm_wiki` source watcher routing](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/project-file-sync.ts)：created/modified/deleted tasks 和 source cleanup routing。
- [`atomicstrata/llm-wiki-compiler` Configurable Lifecycle Profiles](https://github.com/atomicstrata/llm-wiki-compiler/blob/cbd09c6c415f36b6001adf89a02aa805a5a5aba6/docs/concepts/configurable-lifecycle-profiles.mdx)：profile 宣告、typed writes、lifecycle、review/trust gate。
- [`atomicstrata/llm-wiki-compiler` review policy](https://github.com/atomicstrata/llm-wiki-compiler/blob/cbd09c6c415f36b6001adf89a02aa805a5a5aba6/docs/configuration/review-policy.mdx)：候選 queue、hold reason、fail-closed policy。
- [`atomicstrata/llm-wiki-compiler` lifecycle apply](https://github.com/atomicstrata/llm-wiki-compiler/blob/cbd09c6c415f36b6001adf89a02aa805a5a5aba6/src/trust/lifecycle-apply.ts)：under-lock transition validation、evidence、journal/audit ordering。
- [`atomicstrata/llm-wiki-compiler` review approve](https://github.com/atomicstrata/llm-wiki-compiler/blob/cbd09c6c415f36b6001adf89a02aa805a5a5aba6/src/commands/review-approve.ts)：approve 時 re-read/re-validate/re-plan，且不重新呼叫 LLM。
- [`atomicstrata/llm-wiki-compiler` source removal](https://github.com/atomicstrata/llm-wiki-compiler/blob/cbd09c6c415f36b6001adf89a02aa805a5a5aba6/docs/cli/rm.mdx)：`--dry-run`、shared concept、orphan、typed entity source-ownership limitation。

## 28. WikiSkill 論文審計與演進層更新

本節把最新論文研究和本方案的正式調整集中記錄；實作規範以[工程實作規格](<./SME-知識庫工程實作規格.md>)第 15 節為準，論文逐項證據見[WikiSkill 論文第三方審計](<./docs/wikiskill-paper-audit-20260904.md>)。

### 28.1 審計結論

WikiSkill 的合理核心是 `immutable trace → pattern → atomic Skill proposal → validation/history`。論文是 2026-08-27 的 arXiv preprint，沒有足夠證據證明官方 orchestration、完整 benchmark artifact、seed/config、checkpoint/environment lock 和 harness 已公開；因此性能結果不能直接視為可重現或企業知識正確性的保證。

| 主張／設計 | 審計結果 | 本方案處理 |
|---|---|---|
| experience 可編譯成可重用 pattern | 論文實驗內有條件支持 | 採用 redacted experience、pattern、iteration/history 分層 |
| validation `new_score > best_score` 可保證改善 | validation 小、重複使用，未充分證明 | deterministic regression + golden cases + 人工 MR；不以單一 score promotion |
| persistent Wiki 自動變正確 | persistent 不等於 truth maintenance | evolution 與 domain entity/rule 分離；active 仍需 evidence/authority/scope |
| 模型越強 skill 越有價值 | 存在 negative transfer；論文明示 Gemini Spreadsheet 套用 Qwen skill 後下降 | 記錄 model/host/skill compatibility，candidate 必須做 profile-specific regression |
| optimizer call complexity 為 `O(1)` | 只計 optimizer LLM calls，不含 inference/token/validation/tool/human 成本 | 不作成本承諾；CI 完全不呼叫 LLM API |

### 28.2 調整後架構：演進但不自動成為真相

```text
Canonical domain knowledge (Git SSOT)
  + reviewed evolution knowledge (Git, same MR)
  + private redacted raw evidence (.cache, hash-addressed)
  + active executable Skills
          ↑
Copilot session → experience → curator proposal → deterministic checks
          → reviewer checklist → SME/CODEOWNER approval → Maintainer merge
          → post-merge Python rebuild (SQLite current; LanceDB future adapter)
```

新增目錄：

```text
knowledge/evolution/{experiences,patterns,skill-history,iterations}/
.cache/evolution/raw/
```

規範新增四個 schema 契約（實作列入 Phase 5）：`evolution-experience`、`evolution-pattern`、`evolution-iteration` 和 `skill-history`。experience 必須有 `trace_hash`、`source_commit`、`agent_id`、`skill_version`、`redaction_status`、`evidence_refs`；iteration 必須有輸入 trace、pattern/skill diff 和 validation report hashes。`observed → candidate → validated → active` 是正常狀態，`validated` 不等於批准；rejected/superseded/retired 永久保留但排除 active retrieval。

### 28.3 採納、限制與不採納

| WikiSkill／第三方實作的設計 | 採納到本方案 | 明確限制 |
|---|---|---|
| immutable raw trace、redaction、hash | raw 預設 private cache；分享只保存 redacted summary | 不進 runtime、不進 Git，不保存 secret/完整 hidden reasoning |
| pattern pages、iteration/log history | Git evolution records 和 immutable manifests | 不另建第二套 domain truth |
| one atomic Skill proposal、rollback | 單一 MR 中保存 candidate diff、rejection/superseded history | 不做 autonomous proposer 或 CI LLM judge |
| re-read/revalidate、path allow-list、deterministic proof | base hash、scope manifest、dry-run、schema/lifecycle/index checks | proof 不是 semantic approval；仍須 SME/CODEOWNER |
| shared-source/reference cleanup | deterministic relation/link rewrite、redirect/tombstone | 日常 delete 用 retire；purge 受同一 MR policy gate |

### 28.4 更新後 Roadmap

| 階段 | 新增或調整交付 | 完成條件 |
|---|---|---|
| 0–4 | 維持現有 schema、Router、SQLite/MCP、single-MR 基線 | lexical runtime 和 deterministic governance 可用，無 LLM API |
| 5. Evolution capture | redacted experience、四個 schema、hash/secret/path checks、pattern/iteration proposal | Copilot 可起草但不能直接 active；raw 不進 retrieval |
| 6. Skill promotion | skill-history、candidate/stable 指針、golden/regression、cross-model metadata | 同一 MR review 後才 active；rejection/superseded 可回溯 |
| 7. Operations | source health、conflict/duplicate report、build manifest、rollback | build/index/source commit 可對帳；失敗保留上一 valid build |
| Future semantic POC | 一個 pinned local embedding provider + LanceDB candidate build | 只有通過資源、品質、license、consistency 和 rollback gate 才啟用 |

### 28.5 最終判斷

本項目可以吸收 WikiSkill 的「經驗累積和 Skill 演進」能力，但不應複製其實驗性 autonomous promotion。成熟版本的成長定義是：模型或 Skill 變更可重放、可評估、可追溯、可回滾；domain facts 仍由 Git source、deterministic compiler 和人手批准共同守住。這個取捨同時滿足 VS Code GitHub Copilot-only、低複雜度、可審查和長期演進目標。
