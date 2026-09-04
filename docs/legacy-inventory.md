# Legacy / noise inventory

Reviewed 2026-09-03. This is a cleanup record, not an implementation contract. The Chinese engineering specification remains the only normative document.

| Legacy or noisy material | Current disposition | Engineer-facing rule |
|---|---|---|
| Historical calibration tables and repeated architecture summaries | Retained only in the research appendix | Implement only from the engineering specification |
| LLM API extraction, Ragas/LLM-as-judge, hosted reranker, VLM API | Removed from the current pipeline | No CI job calls an LLM API or needs an LLM secret |
| Embedding model installation and vector build | Future capability | Implement the provider interface and `DisabledEmbeddingProvider`; current profile is `core-lexical` |
| LanceDB FTS-only profile | Research option, not baseline | Do not maintain a second lexical index now |
| GraphRAG, Neo4j, Kuzu, Leiden | Comparison/future audit only | Use Graph-lite typed relations projected into SQLite |
| Fixed latency/quality thresholds | Rejected without benchmark evidence | Measure locally; do not hard-code universal gates |
| Two-stage MR, MR-A/MR-B, external pre-authorization | Removed | Every operation, including purge, uses one MR with same-MR approvals |
| Purge ticket/Jira/ServiceNow prerequisite | Removed | Purge requires reason, target allow-list, dry-run, and approvals in the same MR |
| Approval handled by Copilot or CI API calls | Rejected | GitLab approval is a platform event; Copilot cannot approve or merge |
| SQLite/LanceDB committed as canonical files | Rejected | Git Markdown/YAML/Mermaid is SSOT; indexes are disposable |
| Formal JSON schemas claimed as already complete | Corrected | Schema files and validators are the next implementation slice |
| Open questions in research appendices | Retained as context only | Current baseline choices in the normative specification override them |
| WikiSkill autonomous Wiki Maintainer/Skill Proposer | Research-only pattern | Keep redacted experience → proposal → deterministic checks → one MR; no CI LLM call or autonomous promotion |
| Paper claim that persistence automatically improves truth | Unsupported for internal domain knowledge | Separate evolution records from canonical entities/rules; require evidence and SME/CODEOWNER approval |
| Strict `new_score > best_score` as universal gate | Benchmark-specific research choice | Use regression/golden evidence plus human review; allow neutral-but-useful changes when justified |
| Full WikiSkill raw/wiki layout copied into runtime | Adapted only | Use four typed evolution directories and private raw cache; no second truth store |

## Final contradiction rules

- No implementation instruction requires two MRs, Jira/ServiceNow tickets, or an LLM API.
- `core-lexical` is the only current retrieval profile; semantic/hybrid is a reserved interface.
- Purge has one MR, same-MR required approvals, deterministic allow-list/dry-run, and at most a protected execution gate.
