# WikiSkill Paper: Independent Audit

> Audit date: 2026-09-04  
> Paper: **WikiSkill: Compiling Agent Experience into Persistent Knowledge for Skill Evolution**, arXiv `2608.27454v1`, submitted 2026-08-27.  
> This is an evidence companion, not the engineering specification. The [Chinese audit](<./wikiskill-paper-audit-20260904.md>) is the primary version.

## Scope and sources

The audit cross-checked the paper HTML, abstract/metadata, arXiv source archive, OpenAlex, and two independent public implementations. The paper is an arXiv preprint, not peer-reviewed evidence of a production knowledge system.

- [Paper HTML](https://arxiv.org/html/2608.27454v1)
- [Paper abstract/metadata](https://arxiv.org/abs/2608.27454)
- [arXiv source archive](https://arxiv.org/src/2608.27454)
- [OpenAlex record](https://api.openalex.org/works/https://doi.org/10.48550/arxiv.2608.27454)
- [Independent Python implementation](https://github.com/ashutoshsinghpr7/wikiskill/tree/02fac2c804fe156e43b12c691e4ae527614d63a1)
- [Independent compiler implementation](https://github.com/poweredbyGEN/wikiskill/tree/2551d24ab754ac3af984bf761f69fdf5322671b7)

## Method reconstruction

```text
immutable execution traces
        ↓
Wiki Maintainer: up to 8 sampled traces (5 failures, 3 successes; 15,000 chars each)
        ↓
pattern pages + index/log/skill-impact history
        ↓
Skill Proposer: at most one atomic Skill proposal per iteration
        ↓
validation split gate: accept only when new_score > best_score
        ├─ accepted Skill becomes active
        └─ rejected Skill rolls back; Wiki/proposal/rejection history remains
```

During training rollouts, the Inference Agent is restricted from Wiki access; active Skills are injected at inference. The paper's `O(1)` optimizer-call claim applies only to full-batch optimizer calls, not inference, tokens, validation, tools, storage, or human review.

## Reported results and claim audit

| Model | No Skill | WikiSkill | Gain |
|---|---:|---:|---:|
| Qwen-3.5-4B | 26.2 | 38.5 | +12.3 |
| Qwen-3.5-9B | 29.9 | 47.4 | +17.5 |
| Qwen-3.6-27B | 39.4 | 63.3 | +23.9 |
| Gemma-4-31B | 41.3 | 54.9 | +13.6 |
| Gemini-3.5-Flash | 49.5 | 68.1 | +18.6 |

| Claim/design | Independent assessment | Decision for this project |
|---|---|---|
| Traces can compile into reusable patterns | Conditionally supported in the reported experiments | Adopt evidence/pattern/proposal/history layers |
| `new_score > best_score` guarantees improvement | Not established; repeated small validation can overfit and strict `>` rejects neutral utility | Use deterministic regression, golden cases, and human review |
| Persistence automatically makes the Wiki correct | Persistence is not truth maintenance | Keep evolution separate from domain Entity/Rule truth |
| Stronger models always benefit from Skills | False in general; Gemini Spreadsheet fell from 50.5 to 18.1 with a Qwen-4B Skill | Record compatibility and test each model/host profile |
| Optimizer cost is `O(1)` | Narrowly true for full-batch optimizer LLM calls only | Make no end-to-end cost claim; CI calls no LLM API |
| Enterprise deployment is demonstrated | Not reproducible from the public artifact alone | Treat as research evidence, not a deployment guarantee |

## Reproducibility and implementation evidence

The paper provides method descriptions, prompts/roles, splits, tables, and figures, but no complete official orchestration, execution harness, all traces, benchmark artifact, seeds/configuration, checkpoint, or locked environment. The independent implementations are useful engineering evidence, not official validation. One records harmful proposals, stale grading, launch failures, and no-op/rejected live runs; the other adds redaction, immutable fixtures, path allow-lists, deterministic verification, and manifest digests.

## Adaptation to the SME knowledge base

```text
Git domain knowledge (canonical SSOT)
  + reviewed evolution records (same GitLab MR)
  + private redacted raw evidence (.cache, hash-addressed)
  + active executable Skills
        ↑
Copilot session → experience summary → curator proposal
        → deterministic checks → reviewer checklist
        → SME/CODEOWNER approval → Maintainer merge
        → post-merge Python rebuild (SQLite current; LanceDB future)
```

| Adopt | Constrain or reject |
|---|---|
| Immutable/redacted traces, hashes, pattern records, iteration history, atomic Skill proposals, rollback history, re-read/revalidate, path allow-lists, deterministic reference cleanup | No raw trace in runtime or Git; no secrets or hidden reasoning; no autonomous Wiki Maintainer/Proposer; no LLM-as-judge or CI model calls; no automatic Entity merge or Skill activation |

The implementation adds `knowledge/evolution/{experiences,patterns,skill-history,iterations}/` and private `.cache/evolution/raw/`. Records use `observed → candidate → validated → active`; `validated` is not approval. Rejected, superseded, and retired records remain auditable but are excluded from active retrieval. Four schema contracts are specified for Phase 5 implementation.

## Verdict

WikiSkill offers a sound pattern for accumulating agent experience and evolving Skills, but it does not prove that an LLM-maintained Wiki is authoritative domain knowledge, cross-model transfer is stable, or the experiments are fully reproducible. This project should adopt a proposal-first, human-approved evolution layer while retaining Git SSOT, deterministic Python validation/compilation, SQLite FTS5 and Graph-lite retrieval, a read-only MCP, a single GitLab MR, and no LLM API pipeline.

