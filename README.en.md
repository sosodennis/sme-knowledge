# SME Knowledge Base

## Engineer entry point

The Chinese engineering specification is authoritative. Use the English file as a companion translation only:

- [Chinese engineering specification](<./SME-知識庫工程實作規格.md>) — normative SSOT for implementation, testing, and MR acceptance.
- [English engineering specification](<./SME-Knowledge-Base-Engineering-Spec.md>) — maintained English companion; Chinese wins if wording differs.
- [Chinese research calibration and roadmap](<./SME-知識庫調研校準與實現路線圖.md>) — research evidence and alternatives, not requirements.
- [English research calibration and roadmap](<./SME-Knowledge-Base-Research-Calibration-and-Roadmap.md>) — English research companion, not requirements.
- [Verified sources](<./docs/verified-sources.md>) — checked official and pinned-commit references.
- [Skills/Agents deep research](<./docs/skills-agents-research-20260903.md>) — evidence for Skills, Custom Agents, handoffs, guardrails, and pressure tests.
- [Legacy/noise inventory](<./docs/legacy-inventory.md>) — retired directions and their disposition.
- [WikiSkill paper audit](<./docs/wikiskill-paper-audit-20260904.en.md>) — English companion; the [Chinese audit](<./docs/wikiskill-paper-audit-20260904.md>) is primary.

The current core baseline is Git canonical files, Python, SQLite FTS5, Graph-lite relations, read-only local MCP, a VS Code Copilot `sme-router` → bounded-worker proposal workflow, GitLab single-MR governance, and post-merge deterministic rebuild. The evolution layer stores redacted experience, patterns, Skill history, and iteration records, but remains proposal-first and human-approved; raw traces never enter runtime retrieval. Embeddings, LanceDB vector search, GraphRAG, Ragas, and LLM API pipelines are not current implementation dependencies.

Process diagrams and images are reserved by the architecture contract: Mermaid is stored as text plus structured process fields, while raster/vector files use asset sidecars with hashes, provenance, and alt/summary text. Current retrieval is limited to that text metadata; pixel-semantic search is not available. The repository has not yet implemented `knowledge/`, `schemas/`, the compiler, or MCP server, so this becomes runnable only after the Phase 1–3 implementation work in the specification.

Reusable code is stored as `knowledge/snippets/<snippet-id>.md`: frontmatter records language, versions, dependencies, purpose, source, license, test, and security status, while the body preserves one exact fenced code block. Multi-file examples are split into linked snippet records. The initial profile uses SQLite FTS5 for identifiers/imports/API names; `get_snippet` returns controlled plain text and metadata only. The compiler, CI, MCP, and Agents never execute a retrieved snippet. This is an implementation contract, not a claim that the directories, schemas, or runtime are already present; deliver it through Phases 1–3.
