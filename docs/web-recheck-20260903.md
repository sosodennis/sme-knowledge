# Web recheck record

Accessed 2026-09-04. Direct HTTP checks were used because the in-app browser connection timed out during setup; no search-result snippets were used as evidence.

## Primary URL availability

The following primary source groups were checked with `curl -L --max-time 30`; all returned HTTP 200:

- VS Code Agent Skills, Custom Agents, and MCP documentation:
  - <https://code.visualstudio.com/docs/agent-customization/agent-skills>
  - <https://code.visualstudio.com/docs/agent-customization/custom-agents>
  - <https://code.visualstudio.com/docs/agent-customization/mcp-servers>
  - <https://code.visualstudio.com/docs/agents/run/subagents>
  - <https://code.visualstudio.com/docs/agents/concepts/trust-and-safety>
- Agent Skills and MCP specifications:
  - <https://agentskills.io/specification>
  - <https://modelcontextprotocol.io/specification/2026-07-28>
  - <https://modelcontextprotocol.io/specification/2026-07-28/server/tools>
  - <https://github.com/modelcontextprotocol/python-sdk>
- Skills and agent authoring research:
  - <https://raw.githubusercontent.com/github/awesome-copilot/main/docs/README.skills.md>
  - <https://raw.githubusercontent.com/github/awesome-copilot/main/docs/README.agents.md>
  - <https://raw.githubusercontent.com/github/awesome-copilot/main/CONTRIBUTING.md>
  - <https://raw.githubusercontent.com/anthropics/skills/main/README.md>
  - <https://raw.githubusercontent.com/anthropics/skills/main/skills/skill-creator/SKILL.md>
  - <https://raw.githubusercontent.com/obra/superpowers/main/skills/writing-skills/SKILL.md>
  - <https://raw.githubusercontent.com/obra/superpowers/main/skills/verification-before-completion/SKILL.md>
  - <https://raw.githubusercontent.com/wshobson/agents/main/docs/authoring.md>
  - <https://raw.githubusercontent.com/wshobson/agents/main/docs/architecture.md>
  - <https://raw.githubusercontent.com/wshobson/agents/main/docs/harnesses.md>
  - <https://owasp.org/www-project-top-10-for-large-language-model-applications/>
  - <https://www.anthropic.com/engineering/building-effective-agents>
  - <https://docs.langchain.com/oss/python/langchain/multi-agent>
  - <https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/selector-group-chat.html>
  - <https://openai.github.io/openai-agents-python/guardrails/>
- SQLite and LanceDB:
  - <https://www.sqlite.org/fts5.html>
  - <https://www.sqlite.org/lang_with.html>
  - <https://docs.lancedb.com/search/full-text-search.md>
  - <https://docs.lancedb.com/search/hybrid-search.md>
- GitLab governance:
  - <https://docs.gitlab.com/user/project/merge_requests/approvals/>
  - <https://docs.gitlab.com/user/project/codeowners/>
  - <https://docs.gitlab.com/ci/pipelines/merge_request_pipelines/>
- LLM Wiki and lifecycle references:
  - <https://github.com/nashsu/llm_wiki>
  - <https://github.com/atomicstrata/llm-wiki-compiler>
  - <https://www.mediawiki.org/wiki/Wikibase/Merging>
  - <https://www.w3.org/TR/prov-o/>
- Graph references:
  - <https://microsoft.github.io/graphrag/>
  - <https://developers.llamaindex.ai/python/framework/module_guides/indexing/lpg_index_guide/>
  - <https://neo4j.com/blog/genai/what-is-graphrag/>

- WikiSkill paper and independent implementations:
  - <https://arxiv.org/html/2608.27454v1>
  - <https://arxiv.org/abs/2608.27454>
  - <https://arxiv.org/api/query?id_list=2608.27454>
  - <https://arxiv.org/src/2608.27454>
  - <https://api.openalex.org/works/https://doi.org/10.48550/arxiv.2608.27454>
  - <https://github.com/ashutoshsinghpr7/wikiskill>
  - <https://github.com/ashutoshsinghpr7/wikiskill/tree/02fac2c804fe156e43b12c691e4ae527614d63a1>
  - <https://github.com/ashutoshsinghpr7/wikiskill/blob/02fac2c804fe156e43b12c691e4ae527614d63a1/wikiskill/harness.py>
  - <https://github.com/ashutoshsinghpr7/wikiskill/blob/02fac2c804fe156e43b12c691e4ae527614d63a1/docs/RUNS.md>
  - <https://github.com/poweredbyGEN/wikiskill>
  - <https://github.com/poweredbyGEN/wikiskill/tree/2551d24ab754ac3af984bf761f69fdf5322671b7>
  - <https://github.com/poweredbyGEN/wikiskill/blob/2551d24ab754ac3af984bf761f69fdf5322671b7/src/wikiskill/proof.py>
  - <https://github.com/poweredbyGEN/wikiskill/blob/2551d24ab754ac3af984bf761f69fdf5322671b7/README.md>

The Semantic Scholar API was also queried during research but returned HTTP 429 (rate limited) on recheck; it is not counted as a successful source URL.

## Document-wide recheck

All tracked Markdown documents (excluding the ignored `.codex-tasks/` working directory) contained 95 unique HTTPS URLs at the last document-wide scan. One `https://example.invalid/source` is an intentional schema-template placeholder. The other 94 real URLs returned HTTP 200 with `curl -L --silent --show-error --max-time 30` on 2026-09-04. The set includes the WikiSkill paper, OpenAlex metadata, pinned independent implementations, and the existing VS Code, storage, graph, lifecycle, Skills/Agents, routing, and LLM Wiki references. Re-run the scan before release because product and repository URLs can change.

## Findings that affect the baseline

| Area | Evidence | Decision |
|---|---|---|
| VS Code customization | Skills require `SKILL.md`; custom agents use `.agent.md`; subagents support explicit `agents` allow-lists and bounded orchestration | Keep two core Skills, one low-privilege Router, three bounded workers, and a read-only local MCP |
| Routing patterns | Anthropic, LangChain, and AutoGen all describe classification/selection plus explicit termination or candidate limits | Use Router → allow-listed worker transitions, max four hops/one retry per stage, contract checks, and fail-closed escalation; add no orchestration framework |
| MCP SDK | Python SDK v2 is the stable line and supports Python 3.10+ | Pin the SDK major/minor; no model calls |
| SQLite/LanceDB | SQLite FTS5 supports lexical search; LanceDB FTS supports BM25 without embeddings, while hybrid needs a query vector | Use SQLite `core-lexical`; reserve LanceDB for future semantic builds |
| GitLab | Required approvals/CODEOWNERS are enforcement features subject to tier and protected-branch settings | Use one MR; verify target tier in Phase 0 |
| GraphRAG/property graph | Common indexing paths use LLM extraction and/or embeddings | Adopt typed Graph-lite relations and bounded SQLite traversal |
| LLM Wiki | Source traceability, review queues, revalidation and redirect/cleanup patterns are useful | Adopt provenance and deterministic lifecycle; reject direct LLM writes and normal physical deletion |

See the [verified source index](<./verified-sources.md>) for the broader source list and the Chinese research document for detailed comparisons.
