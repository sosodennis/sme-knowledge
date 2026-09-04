# WikiSkill 論文第三方審計

> 審計日期：2026-09-04  
> 審計對象：**WikiSkill: Compiling Agent Experience into Persistent Knowledge for Skill Evolution**，arXiv `2608.27454v1`，2026-08-27。  
> 定位：獨立研究審計，不是本項目的工程規範。工程實作仍以[中文工程實作規格](<../SME-知識庫工程實作規格.md>)為唯一準則。

## 1. 審計範圍與證據

本審計以論文 HTML、abstract、arXiv metadata、source tar、OpenAlex metadata 及兩個公開第三方實作交叉核對。論文目前是 arXiv preprint，沒有同行評審或官方 production implementation 的證據；因此性能數字只代表該論文報告的實驗條件，不能直接當成內部 SME 知識庫的保證。

主要來源：

- [論文 HTML](https://arxiv.org/html/2608.27454v1)
- [論文 abstract/metadata](https://arxiv.org/abs/2608.27454)
- [arXiv source archive](https://arxiv.org/src/2608.27454)
- [OpenAlex record](https://api.openalex.org/works/https://doi.org/10.48550/arxiv.2608.27454)
- [第三方 Python WikiSkill harness](https://github.com/ashutoshsinghpr7/wikiskill/tree/02fac2c804fe156e43b12c691e4ae527614d63a1)
- [第三方 harness source](https://github.com/ashutoshsinghpr7/wikiskill/blob/02fac2c804fe156e43b12c691e4ae527614d63a1/wikiskill/harness.py)
- [第三方 run records](https://github.com/ashutoshsinghpr7/wikiskill/blob/02fac2c804fe156e43b12c691e4ae527614d63a1/docs/RUNS.md)
- [另一個 compiler implementation](https://github.com/poweredbyGEN/wikiskill/tree/2551d24ab754ac3af984bf761f69fdf5322671b7)
- [其 proof validator](https://github.com/poweredbyGEN/wikiskill/blob/2551d24ab754ac3af984bf761f69fdf5322671b7/src/wikiskill/proof.py)

外部方法對照另參考 [Anthropic effective agents](https://www.anthropic.com/engineering/building-effective-agents)、[W3C PROV-O](https://www.w3.org/TR/prov-o/)、[Wikibase merging](https://www.mediawiki.org/wiki/Wikibase/Merging)、[SQLite FTS5](https://www.sqlite.org/fts5.html) 和 [GitLab approvals](https://docs.gitlab.com/user/project/merge_requests/approvals/)。完整來源清單見 [`docs/verified-sources.md`](<./verified-sources.md>)。

## 2. 論文方法重建

論文把「agent 執行經驗」編譯成可持久化的 pattern，再以 pattern 支援 executable skill 的迭代。其概念層次可重建為：

```text
Raw execution traces (immutable)
        |
        v
Wiki Maintainer
  - 每輪抽取最多 8 條 trace
  - 最多 5 failure + 3 success
  - 每條最多 15,000 characters
        |
        v
Wiki layer: pattern pages, index.md, logs.md, skill-impact.md
        |
        v
Skill Proposer (每輪最多一個 atomic skill proposal)
        |
        v
Validation split gate: accept only when new_score > best_score
        |
        +--> accepted active SKILL.md
        +--> rejected skill rollback; wiki/proposal/rejection history retained
```

訓練時的 inference agent 不讀 Wiki；這是為了隔離 skill evolution 實驗，避免把既有 Wiki 當作輸入污染測試。active skill 則在推理時完整注入 system prompt。論文的 full-batch 說法「optimizer LLM call complexity 對 training-set size 為 `O(1)`」只適用於其 optimizer 迴圈，並不包括 inference、validation、工具呼叫、token 數量、儲存或人工審批成本。

## 3. 論文內部實驗摘要

論文報告五個 benchmark、五個模型及三次完整 evolution runs；平均分數如下。這些數字是論文內部報告，不是本審計重新跑出的結果。

| Model | No skill | WikiSkill | 絕對提升 |
|---|---:|---:|---:|
| Qwen-3.5-4B | 26.2 | 38.5 | +12.3 |
| Qwen-3.5-9B | 29.9 | 47.4 | +17.5 |
| Qwen-3.6-27B | 39.4 | 63.3 | +23.9 |
| Gemma-4-31B | 41.3 | 54.9 | +13.6 |
| Gemini-3.5-Flash | 49.5 | 68.1 | +18.6 |

| Benchmark | Train | Validation | Test |
|---|---:|---:|---:|
| LiveMath | 35 | 18 | 124 |
| SealQA | 16 | 10 | 85 |
| SpreadsheetBench | 80 | 40 | 280 |
| OfficeQA | 50 | 24 | 172 |
| ALFWorld | 39 | 18 | 134 |

## 4. Claim-by-claim audit

判定使用 `支持`、`有條件支持`、`未充分證明` 和 `不適用於本項目`。`支持`只表示方法在論文實驗內合理，不表示已證明可在任意模型、資料或企業知識庫重現。

| 論文主張 | 審計判定 | 證據與限制 | 對 SME KB 的決定 |
|---|---|---|---|
| trace → pattern → skill 可以累積可重用經驗 | 有條件支持 | 分層與結果改善在論文設定內一致；pattern 由 LLM 綜合，未必是 domain truth | 採用 evidence/pattern/proposal/history 分層，不讓 pattern 自動成為事實 |
| validation split gate 可保證 skill 改善 | 未充分證明 | validation 集小且被多輪反覆使用；只接受嚴格 `>`，neutral useful change 會被拒絕；未充分交代 seeds、信賴區間及 multiple-comparison 控制 | 不採用自動 promotion；用 deterministic regression + 同一 MR 人工批准 |
| Wiki 持久化後會越來越正確 | 未充分證明 | persistent 只代表保存，沒有獨立真值驗證；錯誤 pattern 可能累積 | evolution record 標明 evidence/status，active 仍需 schema、來源和 SME approval |
| 模型越強，skill 越有價值 | 反例存在 | 論文有 negative transfer；Gemini Spreadsheet baseline 50.5，套用 Qwen-3.5-4B skill 後降至 18.1 | skill 必須有 model/host compatibility metadata 和 regression cases，不能跨模型盲目共享 |
| Wiki access 是主要收益來源 | 有條件支持 | ablation 主要在 Gemini-3.5-Flash 與指定 benchmark；不能外推到 Android SME corpus | 用本地 golden cases 驗證，不把論文 ablation 當本項目證明 |
| full-batch optimizer 對資料規模是 `O(1)` | 精確但容易誤讀 | 只計算 optimizer 的 LLM call 次數，不計 token、推理、validation、工具和人工成本 | 不作成本承諾；本項目目前沒有 CI LLM pipeline |
| 方法可直接部署成企業 evolving knowledge base | 不適用／未證明 | source tar 沒有完整 orchestration、benchmark artifact、seed/config、checkpoint/environment lock 或 execution harness | 只吸收資料分層、追溯和候選 skill 變更；canonical domain facts 不允許 autonomous write |

## 5. 可重現性與工程成熟度審查

### 已提供的部分

- exact method description、prompts/角色概念和 benchmark split 說明。
- source tar 包含 `main.tex`、bibliography 和 figures。
- evaluation table、ablation 方向、三次完整 evolution run 的敘述。

### 缺口

- 沒有完整官方 orchestration code、執行 harness、所有輸入 trace、完整 benchmark artifact、random seed/config、model checkpoint 或可鎖定的 environment manifest。
- 沒有足夠資料證明 negative transfer、validation overfitting 和模型/技能相容性已被系統性控制。
- 第三方 [`ashutoshsinghpr7/wikiskill`](https://github.com/ashutoshsinghpr7/wikiskill) 雖有 Copilot backend、isolated profile、strict gate 和 transcript capture，但截至 pinned run records 主要是 rejection/no-op；README 也記錄 stale grading、launch failure 和 harmful skill gate 問題。這反而說明工程風險真實存在。
- [`poweredbyGEN/wikiskill`](https://github.com/poweredbyGEN/wikiskill) 提供 immutable fixture、path allow-list、deterministic verify、manifest digest、secret redaction 和 private report，但它是獨立實作，不能視為論文官方驗證。

**結論：** 論文的核心工作流有可行的工程直覺，但目前只能視為 research evidence，不足以宣稱完全可重現、跨模型穩定或適合直接管理企業真實知識。

## 6. 與本項目差異

| 面向 | WikiSkill | 本項目 baseline | 必須保留的邊界 |
|---|---|---|---|
| 主要對象 | agent experience / executable skills | Android domain knowledge + agent operational experience | domain facts、experience、active skills 分離 |
| 模型位置 | 研究流程內有 optimizer/maintainer/proposer LLM | 只有 VS Code GitHub Copilot 互動式協助 | GitLab CI 不呼叫 LLM API |
| 寫入方式 | skill validation 通過後更新 active skill | proposal branch → deterministic diff → single MR → human approval | Copilot 不能直接寫 active canonical |
| 知識查詢 | active skill 完整注入 system prompt | SQLite FTS5 + Graph-lite + read-only MCP | 保持 lexical fallback；future semantic 只作候選 |
| 驗證 | benchmark score gate | schema/lifecycle/path/hash/index checks + golden cases | 不用單一 LLM score 取代 SME authority |
| 歷史 | Wiki/log/proposal/rollback | Git history + evolution records + build manifests | rejected/superseded 不可進 active retrieval |
| identity lifecycle | 不是 domain Entity 的完整語義模型 | stable ID、scope、authority、merge/split/retire/purge | Entity 操作由 deterministic apply 和 MR 管理 |

論文中的「training agent 不讀 Wiki」不能直接照搬：本項目的 advisor 必須讀已批准的 canonical domain knowledge。正確做法是把 **domain knowledge**、**agent experience** 和 **executable skill** 放在不同資料流，並分別定義檢索和 promotion 規則。

## 7. 採納／拒絕矩陣

| WikiSkill 設計 | 採納方式 | 不採納或限制 |
|---|---|---|
| immutable raw traces | 以 redacted experience 放 `.cache/evolution/raw/`，保存 hash、source commit 和 agent/model metadata | raw trace 不進 runtime retrieval，不進 Git，不能含 secret 或 hidden chain-of-thought |
| pattern layer | `knowledge/evolution/patterns/`，每個 pattern 連到 evidence refs 和 validation refs | pattern 只能是 candidate/validated；不能覆蓋 entity/rule truth |
| atomic skill proposal | `knowledge/evolution/skill-history/` 記錄 base/candidate hash、proposal、結果和 rejection reason | 不做 autonomous proposer；由 Copilot 在 VS Code 互動式起草 |
| strict validation gate | deterministic schema、lifecycle、golden/regression、scope/hash checks | 不要求 `new_score > best_score` 作唯一門禁；neutral-but-useful 由 SME 判斷 |
| rollback/history | 保留 rejected、superseded、retired iteration 和 Git diff | 不把 rollback 變成模型自行執行的 destructive action |
| wiki index/log | 用 iteration manifest、build manifest、review report 取代非結構化長期 log | 不另建第二套 wiki truth 或常駐 graph database |
| model compatibility | 記錄 model/host/prompt/skill versions，按 profile 做 regression | 不宣稱一套 skill 對所有模型通用 |

## 8. 調整後的演進架構

```text
Git canonical domain knowledge
  knowledge/entities | rules | relations | sources
                + reviewed evolution knowledge
  knowledge/evolution/patterns | skill-history | iterations
                ^
                | same single GitLab MR
Copilot session → redacted experience → curator proposal
                → deterministic schema/lifecycle/diff checks
                → reviewer checklist
                → SME/CODEOWNER approve
                → Maintainer merge protected main
                → post-merge Python rebuild
                ├─ SQLite FTS5/control plane (current)
                └─ LanceDB semantic adapter (future, disabled)
```

### 8.1 最小資料契約

```yaml
experience_id: exp-20260904-0001
task_class: android-ui-state
outcome: failure
error_signature: missing-lifecycle-scope
strategy_summary: "先檢查 repeatOnLifecycle 的 scope，再回答"
trace_hash: sha256:<redacted-trace>
source_commit: <git-sha>
agent_id: sme-router/kb-curator
skill_version: sme-kb-maintenance@<git-sha>
model_metadata: {host: vscode-copilot, model: recorded-if-known}
redaction_status: passed
evidence_refs: [source.android.lifecycle]
validation_refs: []
status: observed
```

```yaml
skill_id: sme-kb-maintenance
proposal_id: prop-20260904-0007
base_skill_hash: sha256:<old>
candidate_skill_hash: sha256:<new>
outcome: rejected
validation_refs: [golden.android.lifecycle.01]
rejection_reason: "cross-model regression not investigated"
supersedes: null
```

```yaml
iteration_id: iter-20260904-0002
input_trace_hashes: [sha256:<trace-1>, sha256:<trace-2>]
pattern_diff_hash: sha256:<pattern-diff>
skill_diff_hash: sha256:<skill-diff>
validation_report_hash: sha256:<report>
source_commit: <git-sha>
compiler_version: 0.1.0
review_status: pending
```

### 8.2 狀態與 promotion

`observed → candidate → validated → active` 是正常路徑；`rejected`、`superseded`、`retired` 是保留歷史但排除 active retrieval 的終態或旁路。只有同一 MR 中的 proposal、deterministic validation、reviewer checklist 和 SME/CODEOWNER approval 全部完成，才可把 pattern 或 skill history 標為 `active`。`validated` 不等於已批准；它只代表機械和指定回歸檢查通過。

### 8.3 Router、worker 和 human 責任

Router 只負責發現 route、handoff、hash、timeout、loop 和 scope drift 問題；curator 產生 proposal；reviewer 檢查證據和 contract；Python/CI 驗證機械正確性；SME/CODEOWNER 才能決定 domain truth 和 skill 是否啟用。任何 worker 失敗、無法驗證、越權或 context loss 都要 fail closed，保留 reason code，不由 Router 自動放寬權限。

## 9. 風險登記

| 風險 | 觸發方式 | 控制 |
|---|---|---|
| 錯誤 pattern 污染 domain truth | agent summary 被誤當作 rule | evolution 與 entity/rule 分目錄；active canonical 仍需 source/evidence/MR |
| validation overfit | 反覆試同一小 validation split | 本地 golden/regression、人工抽樣、保留所有 rejection，禁止單一 score promotion |
| cross-model negative transfer | 把某模型 skill 套到另一模型 | skill/model/host metadata、profile-specific cases、candidate 不自動 active |
| raw trace 洩漏 | transcript 含 token、個資或 hidden reasoning | redaction、hash、allow-list、raw 預設 cache/private、CI secret scan |
| autonomous drift | Router/worker 直接修改 active | agent allow-list、MCP read-only、proposal-only curator、protected main |
| build/index 不一致 | merge 後使用舊 SQLite/LanceDB | merged commit rebuild、manifest hash、consistency check、失敗保留上一個 valid pointer |

## 10. 最終判斷

WikiSkill 的可取核心是：把執行經驗分成 immutable evidence、可追溯 pattern、atomic skill proposal 和可回滾歷史，並以明確驗證門禁控制 active skill。它沒有充分證明「LLM 自己維護的 Wiki 就是真實、長期正確的企業知識庫」，也沒有證明跨模型 transfer 或完全可重現。

因此本項目採用 **WikiSkill-inspired、proposal-first evolution layer**：

```text
canonical domain knowledge
+ reviewed evolution knowledge
+ active executable Skills
+ private redacted raw evidence
+ immutable iteration manifests
+ deterministic compiler/validators
+ single-MR human governance
```

這保留了隨模型能力提升而「可重編譯、可重評估、可重檢索」的成長能力，同時維持可審查、低複雜度、Git SSOT、無 LLM API pipeline 和 VS Code GitHub Copilot-only 的硬性要求。

