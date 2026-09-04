# SME Knowledge Base Research Calibration, Architecture, and Roadmap

> Document status: Research and evidence companion; not an engineering specification.
> Language policy: [The Chinese research document](<./SME-知識庫調研校準與實現路線圖.md>) and the [Chinese engineering specification](<./SME-知識庫工程實作規格.md>) remain authoritative. This English document summarizes and translates the research conclusions; if any wording differs, Chinese wins.

**Research date:** 2026-09-04
**Scenario:** An Android SME knowledge base used through GitHub Copilot Chat/Agent in VS Code, without integrating an LLM API, with reviewable Git changes, low complexity, and long-term evolution.

## 1. Executive summary

The project is feasible. The current delivery must remain deliberately smaller than a full GraphRAG or managed vector platform:

```text
Git canonical knowledge (Markdown/YAML/Mermaid/images/code snippets)
        │
        ▼
Deterministic Python compiler and lifecycle validation
        ├─ SQLite FTS5 control plane (current)
        │    metadata, provenance, status, scope, authority, relations
        └─ LanceDB semantic adapter (future, disabled now)
        │
        ▼
Read-only local MCP stdio gateway
        │
        ▼
VS Code Skills / Router / Worker Agents / GitHub Copilot
```

The current profile is `core-lexical`: SQLite FTS5 plus explicit Graph-lite relations. Embeddings are unavailable, so the runtime injects `DisabledEmbeddingProvider`; no model is downloaded, no fake vector is created, and no LanceDB vector table is built. LanceDB remains a future adapter seam, not a current requirement.

The canonical files in Git are the single source of truth. SQLite, LanceDB, reports, and caches are disposable snapshots generated from a stable chunk IR. Copilot may draft a proposal, but deterministic validation and a single GitLab MR are required before canonical knowledge changes. After merge, a post-merge runner rebuilds from the merged `main` commit; a reviewer does not manually pull a branch to trigger the build.

## 2. Research method and decision labels

Primary sources were checked from VS Code, GitHub, MCP, Agent Skills, SQLite, LanceDB, GitLab, Android, W3C, Wikibase, GraphRAG, LlamaIndex, and pinned GitHub commits. A document-wide HTTP recheck found 81 real HTTPS URLs returning HTTP 200; one intentional `example.invalid` schema placeholder was excluded. See the [web recheck record](<./docs/web-recheck-20260903.md>) and [verified sources](<./docs/verified-sources.md>).

| Label | Meaning |
|---|---|
| **Verified** | Directly described by an official source or reproducible locally |
| **Conditional** | Supported by a protocol/package but dependent on host, version, model, resource, policy, or license |
| **Must implement** | Not automatically supplied by VS Code, MCP, or a database |
| **Not a baseline** | Useful for research or migration but adds unnecessary current complexity or risk |

## 3. Calibrated directions

| Original direction or claim | Calibrated direction | Reason |
|---|---|---|
| Git + Markdown/YAML/Mermaid as knowledge source | Keep as canonical SSOT; ignore derived indexes | Reviewable, diffable, rollback-friendly; Mermaid is text-defined |
| Put the entire knowledge base in `SKILL.md` | Skills contain workflow rules; domain facts remain under `knowledge/` | Skills are discovered and loaded progressively; they are not a corpus store |
| Custom SME Agent | Use a low-privilege `sme-router` plus `.agent.md` worker roles with bounded tools and guided handoffs | VS Code supports custom agents, subagents, and handoffs; routing is delegation, not unattended automation |
| LLM routing/supervisor | Add one Router to classify intent and delegate to an allow-listed worker | Anthropic, LangChain, and AutoGen describe routing/selector patterns; keep transitions, retries, and termination bounded |
| Local FastMCP/stdio server | Prefer the official Python `mcp` SDK and expose typed read-only tools | Local MCP is supported but must be trusted and sandboxed; arbitrary write tools are unsafe |
| Image content guarantees visual Copilot reasoning | Always provide alt text, summary, OCR where verified, and structured fallback | MCP image content does not guarantee a specific Copilot host/model will interpret it |
| LanceDB as a first-day vector system | Reserve the adapter; current `core-lexical` builds SQLite only | Semantic retrieval needs an embedding model and independent resource/license checks |
| Kuzu as the embedded graph baseline | Do not adopt for a new baseline | Its repository is archived/read-only |
| Ragas or an LLM-as-judge merge gate | Use deterministic checks, golden cases, and human review | Evaluator scores are model/data/prompt dependent and require an evaluator service |
| Leiden automatically reorganizes canonical files | Use only as an offline audit/report if ever needed | Clustering does not understand business boundaries and must not move source files |
| Fixed quality/latency thresholds | Measure on the real corpus before setting targets | Hardware, corpus size, chunking, and host behavior change the measurements |
| New models cause automatic geometric growth | Recompile, re-evaluate, and re-index under version control | Model improvement is conditional on evidence, schema, and regression checks |

## 4. Final architecture

### 4.1 Responsibilities by layer

| Layer | Contents | Canonical? | Responsibility |
|---|---|---:|---|
| Knowledge source | Markdown, YAML frontmatter, Mermaid, images, fenced code snippets, source links/copies | Yes | Human-readable, reviewable, reversible facts |
| Deterministic compiler | Parse, validate, normalize, chunk, hash, build indexes | No | Compile source into a reproducible snapshot; never invent facts |
| SQLite control plane | Metadata, FTS5, relations, provenance, lifecycle, build manifest | No | Authoritative filtering by status, scope, authority, and evidence |
| Semantic plane | Future local embeddings and LanceDB | No | Candidate retrieval only; never a replacement for provenance/control data |
| MCP gateway | Typed search/get/related/build-info/asset/snippet tools | No | Stable interoperability boundary; read-only by default |
| Copilot customizations | Instructions, Skills, Custom Agents | No | Query behavior, proposal workflow, role/tool boundaries |
| Governance/CI | Schema checks, link checks, lifecycle checks, CODEOWNERS, golden cases | No | Block invalid changes and record evidence |

### 4.2 Single truth and rebuild model

The shared kernel creates one stable chunk IR containing `chunk_id`, `content_sha256`, source path, locator, modality, and compiler version. SQLite and any future LanceDB snapshot consume the same IR. A build is immutable and keyed by source commit plus compiler/schema/chunking versions. `.cache/current.json` changes only after consistency checks and atomic promotion.

For a personal checkout, the user runs `python scripts/build_indexes.py --profile core-lexical` and the local MCP reads `.cache/current.json`. For a shared internal installation, the post-merge GitLab job may publish a source-commit-named artifact. Artifact distribution is an optimization, not a second truth and not a prerequisite for local rebuild.

### 4.3 Repository shape

```text
android-sme-kb/
├─ .github/{copilot-instructions.md,instructions/,agents/{sme-router,android-advisor,kb-curator,kb-reviewer},skills/}
├─ .vscode/mcp.json
├─ knowledge/{sources,entities,rules,relations,processes,decisions,assets,snippets,proposals,conflicts}/
├─ schemas/*.schema.json
├─ src/sme_kb/{contracts.py,compiler.py,lifecycle.py,proposal.py,embedding.py,retrieval.py,provenance.py,security.py,mcp_server.py}
├─ src/sme_kb/stores/{sqlite_store.py,lancedb_store.py}
├─ scripts/{validate_kb.py,validate_lifecycle.py,apply_proposal.py,build_indexes.py,check_index_consistency.py,inspect_relations.py,benchmark_retrieval.py,report_maintenance.py}
├─ configs/{retrieval.yml,models.yml,policy.yml}
├─ tests/{fixtures/,golden_cases.yml,test_*.py}
├─ .cache/builds/<build-id>/{manifest.json,sqlite/,lancedb/}  # gitignored
├─ pyproject.toml
├─ requirements.txt
├─ requirements-dev.txt
└─ .gitlab-ci.yml
```

### 4.4 Process diagrams and image assets

The design supports diagrams and images at the contract level, but the repository currently contains only specifications and research records. The `knowledge/`, `schemas/`, compiler, and MCP implementation still have to be delivered in Phases 1–3; support here must not be read as an already-running ingest feature.

| Type | Canonical representation | Current retrieval after implementation | Boundary |
|---|---|---|---|
| Mermaid flow/sequence/state diagram | Mermaid source in a `process` record, or a controlled `.mmd` asset | Source text plus structured `steps`, `branches`, and `failure_handling` in SQLite FTS5 | Rendered pixels never replace structured semantics |
| PNG/JPEG/WebP/SVG | `knowledge/assets/<asset-id>/asset.yml` plus a controlled payload | Sidecar title, alt text, summary, and verified OCR/labels | `get_asset` can return a resource link; visual reasoning depends on the host/model |
| Screenshot/scanned diagram | Original plus a human-verified transcription/process record | Search the transcription, nodes, relations, and verified OCR | Without text fallback it is archival evidence only |

Asset validation must enforce repository-relative paths, controlled MIME and magic bytes, size limits, license/provenance, sensitive-data checks, and SVG script/external-reference restrictions. Other binary files such as PDF, audio, and video require the same sidecar and individual MIME allow-list; without a verified transcript they are archival evidence only. The lexical compiler emits `diagram`, `image-metadata`, and verified `ocr` text chunks; it does not read pixels or require Pillow/OCR/model dependencies. Cross-modal search is future-only through one validated image/text `EmbeddingProvider` and an independent LanceDB candidate build. An image is evidence/presentation, not the sole basis for a rule or process.

### 4.5 Code snippets

Reusable, independently citable code belongs in `knowledge/snippets/<snippet-id>.md`, not as a second truth in SQLite or as a large Skill prompt. YAML frontmatter records `language`, `language_version`, `snippet_type` (`example`, `template`, `command`, `config`, `patch`, or `pseudocode`), `purpose`, `when_to_use`, `not_for`, `dependencies`, `framework_versions`, `tested_on`, `source`, `license`, `provenance`, `security_review`, and `review_by`. The body contains exactly one fenced code block so the compiler can preserve the exact bytes and compute `content_sha256`.

The baseline maps one snippet to one independently citable single-file code body. Split multi-file examples into linked snippet records rather than inventing an ungoverned bundle format.

The baseline compiler parses and hashes the record and emits a `modality: code` chunk for SQLite FTS5. It never executes shell, SQL, build scripts, Kotlin/Python, or dependency downloads. CI performs schema/path/license validation, secret and credential scanning, and dangerous-command checks. A language parser/linter/compiler such as `tree-sitter` or a Kotlin compiler is optional and requires an approved toolchain; a failed optional check cannot mark the snippet as tested.

Without embeddings, exact identifiers, imports, API names, and comments remain searchable lexically. `get_snippet` returns exact code together with compatibility, source/hash, tested status, security status, and build ID; if the body comes from a derived chunk, the hash must be reconciled with the Git canonical file/build manifest first, otherwise return an index-inconsistency error. Retrieved code is untrusted data: Agents must not execute it or promote an example/template into a project Rule. Snippet lifecycle uses `create/update/retire/restore/purge` and the same single MR; merge is proposed only when semantics, versions, and licenses are compatible. Relations to Entities/Rules stay in `knowledge/relations/`. Future semantic code search may reuse the existing `EmbeddingProvider` seam and LanceDB candidate build.

## 5. VS Code, Skills, Agents, and MCP

The permanent `copilot-instructions.md` should contain only short global rules: search the knowledge source first, cite IDs and locators, distinguish `candidate`/`active`/`conflict`, and say when evidence is missing. It must not contain the entire Android corpus.

The two core Skills are `sme-kb-retrieve` for cited read-only retrieval and `sme-kb-maintenance` for proposal/lifecycle work. A user normally selects only `sme-router`; it classifies the request and invokes one or more bounded workers. The `sme-kb-maintenance` Skill describes repeatable workflows such as adding a source, drafting an entity/rule proposal, inspecting conflicts, validating, and rebuilding; `sme-kb-retrieve` remains read-only and citation-focused. Detailed schema material belongs in references, not in either always-loaded Skill entry point.

The detailed Skills/Agents evidence, source refs, guardrails, handoff artifact, human-gate matrix, invariants, error taxonomy, and pressure scenarios are maintained in [`docs/skills-agents-research-20260903.md`](<./docs/skills-agents-research-20260903.md>). The Chinese engineering specification remains the implementation authority.

The Router and three worker boundaries are:

| Role | Reads | Writes/authority |
|---|---|---|
| Router | User intent, route policy, worker results, handoff artifacts | No writes; classifies, delegates, checks contracts/loops/timeouts, and escalates |
| Advisor | Read-only MCP search/get/related/asset/snippet/build-info | Answers with evidence; no canonical writes or code execution |
| Curator | Sources, existing records, proposal paths | Drafts proposals only; cannot approve or purge |
| Reviewer | Proposal, scope, source, lifecycle and relation evidence | Produces a checklist; cannot replace human approval |

MCP is a local stdio process configured by `.vscode/mcp.json`. It must restrict paths, MIME types, query limits, timeouts, and output size. It must expose typed `get_snippet` output (exact fenced code plus language, compatibility, source/hash, tested/security status, and build ID) without any execution capability. It must not expose arbitrary SQL, filesystem writes, Cypher, or entity mutations. Other Agents consume typed MCP responses and do not need to know SQLite tables or LanceDB internals. Retrieved code remains untrusted data and is never automatically executed.

### 5.1 Router contract and responsibility split

The Router is a coordinator, not a fourth knowledge authority. It is the
recommended sole daily entry point. Set `user-invocable: false` and `agents: []`
on each worker so the normal picker exposes only the Router while the workers
remain callable through the Router's explicit allow-list:

```yaml
name: sme-router
user-invocable: true
tools: [agent, read]
agents: [android-advisor, kb-curator, kb-reviewer]
```

It classifies to `query`, `maintenance`, `review`, `mixed`, `clarify`, or
`blocked`, then follows this fixed table:

| Intent | Route | Result |
|---|---|---|
| `query` | `android-advisor` | Cited answer or explicit no-evidence/conflict result |
| `maintenance` | `kb-curator → kb-reviewer` | Proposal/diff and reviewer status, then the same-MR human gate |
| `review` | `kb-reviewer` | Checklist only; never approval |
| `mixed` | `android-advisor → kb-curator → kb-reviewer` only when the user explicitly requests a change after evidence gathering | Proposal and reviewer status |
| `clarify` | no worker | Ask for operation, target ID, scope, or output format |
| `blocked` | no worker | Preserve reason code and escalate |

Each request is bounded to four hops and one retry per stage. A
`changes-required` report may return to the same curator once; there is no
recursive worker delegation. Because VS Code subagent calls are stateless,
the Router must pass a complete context envelope (request, operation, exact
IDs, source commit, hashes, scope, artifact, and expected output fields).
Before advancing, it checks status, IDs, hashes, scope, validation, and
visited-agent state. Missing/contradictory output, timeout, context loss, an
unlisted worker, or a privilege attempt is a visible `blocked` failure, not a
silent repair.

The Router is responsible for discovering and containing orchestration/process
problems; workers remain responsible for their domain role; deterministic
Python/CI checks remain responsible for mechanical correctness; and
SME/CODEOWNER remains the only semantic approval authority. Router confidence
or status is never evidence or approval. If routing is unavailable, the
Customizations editor may temporarily expose a worker for manual fallback,
without changing its read/write boundary or the single-MR workflow.

## 6. Schema design status

The Chinese normative specification now contains the MVP field-level registry. The next implementation slice creates the executable JSON Schema files and fixtures in this order:

```text
common
  → source/entity/rule/relation
  → proposal/conflict
  → process/decision/asset/snippet/taxonomy
  → chunk-ir/build-manifest
  → evolution-experience/evolution-pattern/evolution-iteration/skill-history
```

The schemas must enforce globally unique non-reusable IDs, controlled enums, active evidence, relation endpoints, field-level proposal patches, lifecycle operation enums, exact single-fenced-body snippet records, and the four Phase 5 evolution contracts. The repository did not previously contain these schema files; they must be implemented and validated rather than assumed to exist.

## 7. Embedding and LanceDB findings

### 7.1 Current no-embedding profile

The absence of an embedding model is not a blocker. SQLite FTS5 provides offline lexical retrieval and BM25-style ranking. Graph-lite relations provide explicit multi-hop context. The MCP response reports `retrieval_mode: lexical` and a semantic-disabled flag.

LanceDB can technically run its own BM25 full-text search without embeddings, but enabling it now would duplicate the SQLite lexical index and create a second operational path. Therefore the current baseline does not build a LanceDB table. A LanceDB FTS-only profile may be evaluated later only if a concrete LanceDB-specific feature or benchmark justifies the duplication.

### 7.2 Future provider seam

```text
EmbeddingProvider
  ├─ available()
  ├─ embed_documents(texts)
  ├─ embed_query(text)
  ├─ provider_id / model_id
  └─ dimension

DisabledEmbeddingProvider
  ├─ available() = false
  └─ embed_* raises EmbeddingUnavailable("embedding_model_not_configured")
```

When a local model becomes available, select one pinned provider, record model revision/dimension/checksum/license, build a candidate LanceDB snapshot, and compare it against the lexical golden set. Only then may semantic or hybrid mode become a default. Vector results remain candidates; SQLite still verifies status, scope, authority, and provenance.

## 8. Graph approaches and the Graph-lite choice

Microsoft GraphRAG extracts entities, relationships, claims, summaries, communities, and embeddings through an LLM-heavy transformation pipeline. Neo4j-style GraphRAG starts with vector/full-text seeds and traverses typed paths. LlamaIndex `PropertyGraphIndex` can orchestrate graph construction and retrieval, but its common extraction path also uses an LLM; its implicit-path pattern is useful when relationships already exist.

| Approach | Strength | Cost/constraint | Decision here |
|---|---|---|---|
| GraphRAG | Global/local/DRIFT multi-hop views and community summaries | LLM extraction, embeddings, Parquet/vector pipeline, prompt tuning | Research comparison only |
| Neo4j/other graph DB | Rich traversal, path filters, graph analytics | New service/runtime, deployment and another truth projection | Future POC only |
| LlamaIndex property graph | Pluggable retrievers and graph orchestration | Framework scope and usual LLM extraction path | Pattern reference only |
| Graph-lite + SQLite | Typed edges, bounded traversal, provenance, no extra service | Manual/canonical relation maintenance; limited analytics | Current recommendation |

Graph-lite keeps typed `source_id → predicate → target_id` edges in Git, compiles them to SQLite, and uses bounded recursive CTE traversal (depth 0–2). Search finds lexical seeds today and may find vector seeds in the future; traversal and authority/provenance filtering remain unchanged. This captures the explainability and multi-hop advantages without adding a graph database or GraphRAG API dependency.

## 9. Entity lifecycle and conflict handling

Entity identity must be stable and reviewable. The lifecycle is:

| Operation | Required behavior |
|---|---|
| Create | New ID, definition, scope, authority, source; candidate until approved |
| Update | Field-level patch with current `base_hash`; stale base fails |
| Retire | Preserve history and reason; exclude from active search |
| Merge | Explicit survivor, loser redirects/tombstones, alias map, deterministic relation rewrite |
| Split | New IDs, field allocation, and destination for every relation; uncertain edges become conflicts |
| Restore | Git revert or reverse proposal; check relations added during the interval |
| Purge | Destructive, allow-listed removal of body/attachments/derived rows; minimum non-sensitive tombstone where policy permits |

Similarity or lexical matching can create duplicate candidates, but cannot automatically merge entities. Adjudication compares definition, scope, authority, aliases, relations, and evidence. An open conflict is query-visible and prevents the Agent from presenting a guessed single answer.

The design adopts the useful Wikibase patterns of retaining a merge survivor and redirecting the losing identity, and the PROV-O patterns of recording derivation, revision, source, generation, and invalidation. It does not introduce RDF or a Wikibase runtime.

## 10. Single-MR approval and rebuild workflow

```text
Copilot/engineer drafts proposal
  → scope_manifest + deterministic canonical diff
  → one Draft MR while preparing
  → MR CI: schema/lifecycle/dry-run/hash/allow-list checks
  → SME/CODEOWNER required approval in that same MR
  → Maintainer merges protected main
  → post-merge runner checks out merged commit
  → rebuild SQLite core-lexical snapshot and consistency check
  → publish immutable job artifact if the shared environment needs it
```

The scope and final result are reviewed once because both are locked in the same MR. `scope_manifest` contains operation, target IDs, allowed paths, expected file count, and `scope_hash`; `purge_manifest` narrows destructive paths and IDs. A purge may additionally use a protected manual execution job for physical cache/artifact cleanup, but that is not a second review or MR.

GitLab required approvals and CODEOWNERS are hard gates only when the target tier and protected-branch settings enforce them. Phase 0 must verify this. If the tier offers only optional approvals, use an equivalent protected-branch and Maintainer checklist/manual gate before claiming enforcement.

## 11. Comparison with LLM Wiki implementations

The inspected `nashsu/llm_wiki` commit demonstrates source traceability, ingest records, duplicate candidates, review storage, source lifecycle cleanup, and page merge/delete mechanics. The inspected `atomicstrata/llm-wiki-compiler` commit demonstrates typed lifecycle profiles, review policies, dry-runs, and re-read/revalidate/re-plan behavior before approval.

| Pattern observed | Adopt | Reject or adapt |
|---|---|---|
| Source traceability and locators | Yes; required on active records and chunks | — |
| Duplicate detection and review queue | Yes; candidate proposal only | No similarity-only automatic merge |
| Re-read/revalidate before approval | Yes; `base_hash`, clean checkout, deterministic dry-run | — |
| Shared-source cleanup | Yes; relation/orphan scan in compiler | Do not silently delete canonical entities |
| Physical losing-page deletion | No for normal lifecycle | Use merged redirect/tombstone; reserve purge for policy-controlled cleanup |
| LLM provider and API-centric ingest | No in current project | Copilot may draft interactively; GitLab CI remains API-free |
| Desktop/HTTP backend | No | Use VS Code + read-only local MCP boundary |

## 12. Dependencies and platform checks

### Current runtime

| Dependency | Role | Current status |
|---|---|---|
| Git | SSOT, branch/MR, rollback | Required |
| Python 3.10+ | Compiler, validator, MCP | Required |
| Official `mcp` Python SDK v2 | Typed local stdio gateway | Required; pin major/minor |
| Pydantic | Typed DTOs and input validation | Required |
| PyYAML | Safe frontmatter parsing | Required; use `safe_load` |
| jsonschema | Executable schema validation | Required |
| Python `sqlite3` with FTS5 | Metadata, lexical search, relations | Required; probe `ENABLE_FTS5` |
| pytest | Tests/golden cases | Dev/CI required |
| ruff | Formatting/lint | Recommended |
| Node + Mermaid CLI | Diagram render | Only if render CI is enabled |
| Language-specific parser/linter (for example `tree-sitter` or a Kotlin compiler) | Optional snippet syntax/AST checks | Future-only; requires an approved toolchain and must not execute arbitrary snippets |

### Future-only dependencies

LanceDB, one pinned local embedding provider (`sentence-transformers` or a verified `fastembed` model), a local reranker, Pillow/OCR, or `leidenalg`/`python-igraph` are capability-gated. They must not be added to the current runtime lock solely because the packages are available. Neo4j, Kuzu, Milvus, GraphRAG, LightRAG, Ragas, LLM-as-judge, hosted embeddings/rerankers, OpenAI/Anthropic SDKs, batch LLM extraction, and VLM APIs are explicitly excluded.

Phase 0 also checks VS Code/MCP trust, Python interpreter selection, GitLab tier/approval enforcement, dependency licenses, and whether local derived artifacts are permitted under data policy.

## 13. Roadmap

| Phase | Deliverable | Exit criteria |
|---|---|---|
| 0. Skeleton/environment | Python package, dependency lock, MCP config, protected branch/CODEOWNERS, FTS5 probe | Clean checkout validates; disabled embedding contract passes; governance settings recorded |
| 1. Schema/seed | Executable schemas, fixtures, 10–20 sources, 20–40 reviewed rules/entities, and at least one reviewed snippet | IDs/evidence/scope/review dates pass; snippet has one fenced body, dependency/test/security metadata, and no secrets |
| 2. Copilot workflow | Two core Skills, `sme-router`, and advisor/curator/reviewer worker Agents | Users normally select only the Router; fixed routes, hop/retry limits, worker contract checks, and handoff artifacts are visible; advisor is read-only, curator proposal-only, reviewer cannot approve, conflicts disclosed |
| 3. Compiler/SQLite/MCP | Stable IR, lifecycle apply, SQLite FTS5, fenced-snippet compilation, `get_snippet`, consistency checker, read-only MCP | Local and post-merge rebuilds are reproducible; code chunks and exact snippet retrieval work; cache can be deleted and rebuilt; no code execution occurs |
| 4. Lifecycle/governance | Merge/split/retire/restore/purge fixtures, base hashes, redirects, scope manifests, single-MR rules | Invalid references, stale updates, unsafe purge scopes, and redirect cycles fail closed |
| 5. Evaluation/operations | Golden cases, source-hit/recall@k, latency, source health, stale/conflict reports, rollback runbook | Comparable lexical baseline and immutable manifests exist |
| Future semantic POC | One pinned local provider, candidate LanceDB snapshot, lexical/hybrid benchmark | Semantic mode is enabled only after resource, quality, consistency, license, and rollback gates pass |

## 14. Final feasibility verdict

The design meets the stated goals with a controlled scope:

1. It runs through VS Code GitHub Copilot Chat/Agent rather than a custom model API.
2. It keeps Git Markdown/YAML/Mermaid as an auditable SSOT.
3. It uses one small Python package, SQLite FTS5, Graph-lite relations, and a read-only MCP boundary.
4. It can improve when model capability becomes available without changing canonical IDs, lifecycle, provenance, or approval contracts.
5. Other MCP-capable Agents can consume the same typed retrieval contract without learning internal database details. The Router is a coordination convenience, not a new source of truth or approval authority.

There are no unresolved current high-level architecture choices. Remaining Phase 0 checks and the formal schema/validator/build implementation are execution work, not alternative designs.

## 15. Source index

The complete URL evidence and access records are maintained in [`docs/verified-sources.md`](<./docs/verified-sources.md>) and [`docs/web-recheck-20260903.md`](<./docs/web-recheck-20260903.md>). The [legacy inventory](<./docs/legacy-inventory.md>) lists retired or non-baseline directions. The [Chinese engineering specification](<./SME-知識庫工程實作規格.md>) is the only normative document.

## 16. WikiSkill paper audit and evolution-layer update

The normative implementation of this extension is in section 15 of the [engineering specification](<./SME-Knowledge-Base-Engineering-Spec.md>). The claim-by-claim evidence is in the [English WikiSkill paper audit](<./docs/wikiskill-paper-audit-20260904.en.md>); the [Chinese audit](<./docs/wikiskill-paper-audit-20260904.md>) is primary.

### 16.1 Audit conclusion

WikiSkill's defensible core is `immutable trace → pattern → atomic Skill proposal → validation/history`. The paper is an arXiv preprint dated 2026-08-27. It does not provide enough evidence of a complete official orchestration, benchmark artifact, seeds/configuration, checkpoints/environment lock, or execution harness. Its performance numbers therefore cannot be treated as fully reproducible or as proof of enterprise knowledge correctness.

| Claim/design | Audit result | Our treatment |
|---|---|---|
| Experience can compile into reusable patterns | Conditionally supported within the paper's experiments | Adopt redacted experience, pattern, iteration, and history layers |
| `new_score > best_score` guarantees improvement | Not sufficiently established; small validation and repeated reuse can overfit | Deterministic regression, golden cases, and human MR; no single-score promotion |
| A persistent Wiki becomes correct automatically | Persistence is not truth maintenance | Separate evolution from domain entities/rules; active records still need evidence, authority, and scope |
| Stronger models always benefit from Skills | Negative transfer exists; the paper reports a Gemini Spreadsheet drop after applying a Qwen Skill | Record model/host/Skill compatibility and run profile-specific regression |
| Optimizer complexity is `O(1)` in training-set size | Counts optimizer LLM calls only, not inference, tokens, validation, tools, or human cost | Make no cost promise; CI has no LLM API |

### 16.2 Updated architecture

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

New directories are `knowledge/evolution/{experiences,patterns,skill-history,iterations}/` and `.cache/evolution/raw/`. Four schema contracts are added to the specification (implementation is Phase 5): `evolution-experience`, `evolution-pattern`, `evolution-iteration`, and `skill-history`. Normal state is `observed → candidate → validated → active`; `validated` is not approval, and rejected/superseded/retired records remain historical but are excluded from active retrieval.

### 16.3 Adoption and constraints

| WikiSkill/third-party design | Adopt here | Constraint |
|---|---|---|
| Immutable raw traces, redaction, hashes | Private raw cache and shareable redacted summaries | No runtime retrieval or Git commit of raw traces; no secrets or full hidden reasoning |
| Pattern pages and iteration/log history | Git evolution records and immutable manifests | No second domain-truth database |
| One atomic Skill proposal and rollback | Candidate diff plus rejection/superseded history in one MR | No autonomous proposer or CI LLM judge |
| Re-read/revalidate, path allow-list, deterministic proof | Base hash, scope manifest, dry-run, schema/lifecycle/index checks | Proof is not semantic approval; SME/CODEOWNER remains required |
| Shared-source/reference cleanup | Deterministic relation/link rewrite and redirects/tombstones | Normal deletion is retire; purge is policy-gated in the same MR |

WikiSkill does not change the domain lifecycle rules. It adds a separate, proposal-only evolution vocabulary:

| Evolution operation | Allowed artifact | Explicitly prohibited |
|---|---|---|
| `record_experience` | Redacted, hash-addressed experience summary | Raw transcript, secret, hidden chain-of-thought, or active fact |
| `propose_pattern` | Pattern candidate with evidence and uncertainty | Direct activation or Entity/Rule overwrite |
| `propose_skill_change` | Skill diff plus iteration/history manifest | Routing-permission changes, direct activation, or MR bypass |
| `validate_evolution` | Schema, hash, regression, and golden-case report | Treating `validated` as approved/active |
| `activate_evolution` | Post-merge/build active projection | A Copilot, Router, or CI pre-merge write |
| `reject_evolution` / `supersede_evolution` | Traceable historical outcome and replacement link | Deleting the reason or hiding the failed candidate |

The curator may draft these records and the reviewer may check them. Only the same-MR SME/CODEOWNER approval, CI pass, and protected Maintainer merge can make an evolution record `active`; `activate_evolution` is a post-merge projection, not an Agent or MCP write tool. If an evolution change also modifies a domain Entity, Rule, or Relation, the normal lifecycle proposal, evidence, base hash, and deterministic mapping remain mandatory in that one MR.

### 16.4 Updated roadmap

| Phase | Added or changed deliverable | Exit criteria |
|---|---|---|
| 0–4 | Keep the existing schema, Router, SQLite/MCP, and single-MR baseline | Lexical runtime and deterministic governance work with no LLM API |
| 5. Evolution capture | Redacted experience, four schemas, hash/secret/path checks, pattern/iteration proposals | Copilot can draft but cannot activate; raw traces never enter retrieval |
| 6. Skill promotion | Skill history, candidate/stable pointers, golden/regression cases, cross-model metadata | Activation only after same-MR review; rejection/supersession is traceable |
| 7. Operations | Source health, conflict/duplicate reports, build manifests, rollback | Build/index/source commit reconcile; failures retain the previous valid build |
| Future semantic POC | One pinned local embedding provider and candidate LanceDB build | Enable only after resource, quality, license, consistency, and rollback gates |

### 16.5 Final judgment

The project can absorb WikiSkill's experience-accumulation and Skill-evolution ideas without copying its experimental autonomous promotion. Mature growth means model or Skill changes are replayable, measurable, traceable, and reversible; domain facts remain protected by Git source, deterministic compilation, and human approval. This preserves the VS Code GitHub-Copilot-only constraint, low complexity, reviewability, and long-term evolution.
