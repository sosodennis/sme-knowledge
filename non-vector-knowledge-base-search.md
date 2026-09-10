# Non-Vector Self-Evolving Knowledge Base Architecture Design: Cognitive Memory Engineering Driven by Symbolic Systems, Graph Topologies, and LLM Compilers

## 1. System Paradigm Shift: From Vector Geometric Retrieval to Neuro-Symbolic Compilation

Traditional Retrieval-Augmented Generation (RAG) fundamentally relies on dense embeddings and Approximate Nearest Neighbor (ANN) search. However, this mechanism exposes intrinsic limitations in enterprise-grade multi-hop reasoning and dynamic knowledge evolution:

* **Geometric Cosine Similarity $\neq$ Strict Logical Relevance:** High-dimensional continuous proximity captures superficial semantic affinities rather than deterministic logical necessity.
* **Context Fragmentation:** Fixed-size chunking inherently breaks semantic coherence when handling strict identifiers, dependency hierarchies, causal chains, and temporal factual mutations.
* **Probabilistic Ouster:** Probabilistic recall lacks interpretability, deterministic provenance, and operational auditability.

By tightly coupling the unstructured semantic comprehension of Large Language Models (LLMs) with the deterministic structures of symbolic systems (knowledge graphs, inverted search indexes, Abstract Syntax Trees (AST), and rule engines), the **"No-Vector DB" Self-Evolving Architecture** transitions the LLM's role:

> **Core Philosophy:** Shift the LLM from a *fuzzy runtime semantic matcher* to an offline **Knowledge Compiler** and a deterministic runtime **Logical Interpreter**.

During ingestion, the LLM compiles unstructured prose into strongly coupled symbolic artifacts. During retrieval and inference, the engine performs deterministic path resolution via inverted indexes, graph traversal, and directory hierarchy routing, deferring to the LLM solely for contextual synthesis. This eliminates context drift and black-box retrieval failures.

### Architectural Paradigm Comparison

| Dimension | Traditional Vector RAG | Neuro-Symbolic Architecture (No-Vector DB) |
| --- | --- | --- |
| **Underlying Representation** | High-dimensional continuous dense vector space | Discrete symbolic structures (Markdown AST, bidirectional link graphs, SPO triples) |
| **Core Retrieval Mechanism** | Metric-space ANN search ($k$-NN / HNSW cosine similarity) | Exact symbolic hit (BM25 / FTS5) + Graph topology traversal (PPR) + Tree-based routing |
| **Multi-Hop Reasoning** | Fragile; heavily dependent on chunk overlap and recurrent vector queries | Resilient; deterministic sub-graph expansion and hierarchical AST directory drill-down |
| **Factual Updates & Conflicts** | Requires embedding re-computation; stale vectors cause silent hallucinations | Deterministic symbol overwrite, bi-temporal versioning, or epistemic branch isolation |
| **Explainability & Audit** | Black-box metric similarity; non-deterministic execution paths | Deterministic citation chains, explicit AST graphs, and immutable Git commit logs |
| **Compute & Storage Footprint** | High GPU inference cost, memory-intensive vector index persistence | Lightweight relational/file systems, high text compression, pure CPU inverted indexing |

---

## 2. In-Depth Deconstruction of the Three Core Implementation Patterns

### Pattern A: Two-Tier Hierarchical "LLM Router + Markdown Index Tree"

Pattern A implements an LLM Wiki architecture and hierarchical document navigation paradigm, bypassing arbitrary text chunking. Documents are maintained as self-contained Markdown files navigated via multi-level LLM agents mimicking technical human researchers.

* **Atomic Document Layer:** Standard Markdown cards with strict YAML frontmatter metadata specifying UUIDs, domain categories, key entities, semantic versions, explicit dependency vectors, and temporal decay weights.
* **Dynamic Directory Navigation Layer:** Global index structures (root `index.md` or strict `llms.txt` specifications) storing high-level domain digests and navigational topologies.

#### Retrieval & Self-Evolution Workflow

During query resolution, an Agent loads the lightweight directory map and executes an **Adaptive Traversal** using two primitive operations: `list_children(node)` and `read_leaf(node)`. If child node descriptions are highly discriminative, the Agent drills down sequentially. If semantics are broad, it triggers **Dynamic Flattening** to scan all underlying leaf summaries.

Knowledge precipitation acts as a compilation pipeline:

1. Ingestion triggers background workers where an LLM derives summaries and asynchronously regenerates `index.md` and `llms.txt`.
2. When multiple atomic cards are repeatedly co-retrieved across reasoning traces, the LLM abstracts higher-order synthesis cards, enabling organic bottom-up structural evolution.

---

### Pattern B: Pure Symbolic Graph & Relational Recursive Retrieval

Pattern B eliminates vector computation by grounding ingestion and query resolution entirely in Entity-Relation-Entity (SPO) triples and graph path topologies, fully executable within zero-dependency embedded relational databases (such as SQLite) or embedded graph engines (such as Kùzu or LatticeDB).

#### Storage Schema (SQLite DDL)

```sql
CREATE TABLE episodes (
    id          TEXT PRIMARY KEY,
    content     TEXT NOT NULL,
    source      TEXT,
    recorded_at TEXT NOT NULL,
    metadata    TEXT
);

CREATE TABLE entities (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    summary     TEXT,
    created_at  TEXT NOT NULL,
    metadata    TEXT
);

CREATE TABLE edges (
    id          TEXT PRIMARY KEY,
    source_id   TEXT NOT NULL REFERENCES entities(id),
    target_id   TEXT NOT NULL REFERENCES entities(id),
    relation    TEXT NOT NULL,
    fact        TEXT,
    valid_from  TEXT,
    valid_until TEXT,
    recorded_at TEXT NOT NULL,
    confidence  REAL DEFAULT 1.0,
    metadata    TEXT
);

CREATE INDEX idx_edges_traversal ON edges(source_id, target_id, valid_until);

```

#### Deterministic Graph Traversal (Recursive CTE)

The retrieval pipeline pairs **Exact Entity Anchoring** via SQLite FTS5 (full-text keyword and alias lookup) with **Recursive Common Table Expressions (CTE)** for multi-hop expansion:

```sql
WITH RECURSIVE graph_traversal(entity_id, depth) AS (
    SELECT ?1 AS entity_id, 0 AS depth
    UNION
    SELECT 
        CASE WHEN e.source_id = gt.entity_id THEN e.target_id ELSE e.source_id END,
        gt.depth + 1
    FROM graph_traversal gt
    JOIN edges e ON (e.source_id = gt.entity_id OR e.target_id = gt.entity_id)
    WHERE gt.depth < ?2 
      AND e.valid_until IS NULL
)
SELECT DISTINCT ent.id, ent.name, ent.entity_type, ent.summary, gt.depth
FROM graph_traversal gt
JOIN entities ent ON ent.id = gt.entity_id
ORDER BY gt.depth ASC;

```

#### Performance Thresholds & Topological Optimization

* **SQLite Scale:** For sparse graphs under 100,000 entities, SQLite Recursive CTE executes 1- to 2-hop subgraph expansions in sub-millisecond to low-millisecond latencies.
* **Graph Explosion:** In power-law networks past 3 hops, intermediate temporary tables risk Cartesian explosion. Migrating to native embedded graph engines featuring Worst-Case Optimal Joins (WCOJ, e.g., Kùzu) or bitset-tracked traversal drops latency by 1–2 orders of magnitude.
* **Noise Suppression (HippoRAG Integration):** To prevent high-indegree hubs from flooding the context, the system runs **Personalized PageRank (PPR)** seeded by anchored entities, dampening irrelevant hub paths and feeding only topologically proximate subgraphs into the LLM context.

---

### Pattern C: Git-Native Knowledge Compilation & Review Pipeline

Pattern C adapts software engineering version control into knowledge management: the knowledge base is treated as an immutable **Codebase**, where all updates are transacted as Git commits or Pull Requests.

```
Incoming Unstructured Data
           │
           ▼
   LLM Semantic Diff (Unified Diff)
           │
           ▼
    CI Static Validation (AST & DAG)
    ├── Valid: Auto-merge to `main`
    └── Conflict: Epistemic Branch Isolation ──► Human-in-the-Loop Review

```

1. **Deterministic Patch Generation:** The LLM generates standard unified diff patches against historical Markdown files rather than performing destructive writes.
2. **CI Pipeline Static Analysis:** An AST pipeline (using `unified`, `remark`, `mdast`) enforces:
* **Bidirectional Link Integrity:** Verifies all `[[wikilinks]]` resolve, eliminating dangling references.
* **Acyclic Hierarchy Check:** Ensures directory categorization constructs a Directed Acyclic Graph (DAG) without circular dependencies.
* **Schema Validation:** Strict YAML frontmatter typing.


3. **Epistemic Branching:** If new input contradicts established facts with low system confidence, the pipeline generates an isolated branch (`epistemic/topic-variant`), preventing silent semantic pollution of the production branch until confirmed by corroborating evidence or manual review.

### Implementation Pattern Trade-off Matrix

| Evaluation Metric | Pattern A: LLM Router + Markdown Tree | Pattern B: Pure Symbolic Graph (SQLite/Graph) | Pattern C: Git-Native Compilation Pipeline |
| --- | --- | --- | --- |
| **Storage Substrate** | Plain text directory, Markdown, `llms.txt` | SQLite database file / Embedded graph binary | Local/Remote Git Repository |
| **Retrieval Engine** | Agent decision tree + Leaf node streaming | FTS5 full-text search + Recursive CTE / PPR | Inverted indexing + Static AST parser + Git log audit |
| **Multi-Hop Depth** | Limited by context window and tree depth (optimal for 2–3 levels) | High; precise graph topology expansion across 3–5 hops | Explicit reference chaining; optimal for DAG provenance |
| **Auditability & Rollback** | Basic file-level snapshot / External logs | Medium; relies on bi-temporal query filtering | Absolute; native SHA cryptographic hashing & atomic rollbacks |
| **Runtime Compute Cost** | Incurs LLM agent reasoning tokens during routing | Zero runtime LLM overhead; pure database-level execution | High CI overhead (AST parsing + semantic alignment) |
| **Self-Healing Mechanism** | Triggered global/local directory re-indexing | Dynamic edge weight decay + orphaned node GC | Automated AST link repair + Epistemic PR merging |

---

## 3. Theoretical Breakthroughs & Algorithmic Design

### 3.1 Entity Resolution & Canonicalization

Without continuous embedding spaces for fuzzy semantic matching, entities are vulnerable to fragmentation (e.g., "Kubernetes", "k8s", and "Kube" mapping to isolated nodes). A four-tier deterministic pipeline solves this:

1. **Deterministic Normalization:** Incoming terms match against a global Alias Mapping Table integrated with domain dictionaries and orthographic converters (e.g., OpenCC for Traditional/Simplified Chinese).
2. **Lexical Trigram Coarse Filtering:** Unmatched entities are scored via SQLite FTS5 Trigram tokens using normalized Levenshtein distance:

$$\text{Sim}_{\text{Lev}}(s_1, s_2) = 1 - \frac{\text{Lev}(s_1, s_2)}{\max(\vert{}s_1\vert{}, \vert{}s_2\vert{})}$$


3. **Topological Structural Matching:** For intermediate similarity ($0.75 \le \text{Sim}_{\text{Lev}} < 0.95$), candidate neighbor sets $\mathcal{N}(u)$ and $\mathcal{N}(v)$ are evaluated using the Jaccard coefficient:

$$J(u, v) = \frac{\vert{}\mathcal{N}(u) \cap \mathcal{N}(v)\vert{}}{\vert{}\mathcal{N}(u) \cup \mathcal{N}(v)\vert{}}$$



High topological overlap triggers automatic alias merging.
4. **LLM Graph Proofreader:** Gray-zone candidates are dispatched to an LLM arbitrator that inspects context paths and emits structured instructions: `Alias Merge`, `Hypernym/Hyponym Link`, or `Separate Entities`, writing decisions back to the alias mapping table.

### 3.2 Truth Maintenance System (TMS) & Dynamic Conflict Resolution

Blindly overwriting old information breaks context (e.g., *"Library X lacks async support"* was true in 2023, but invalid in 2026; yet remains valid for legacy runtime maintainers). The system incorporates a **Justification-Based Truth Maintenance System (JTMS)**:

* **Belief States:** Proposition nodes are marked `IN` (believed true) or `OUT` (lacking valid justification).
* **Justification Tuples:**

$$\text{Justification} = \langle \text{IN-List}, \text{OUT-List} \rangle$$



A justification holds if and only if every node in its $\text{IN-List}$ is `IN`, and every node in its $\text{OUT-List}$ is `OUT`. When premises are revoked, **Dependency-Directed Backtracking** cascades across dependency edges, updating downstream conclusions to `OUT` without recomputing unaffected assertions.

#### Bi-Temporal Binding

Edges carry dual time dimensions: **Valid Time** (`valid_from`, `valid_until`) denoting physical reality validity, and **Transaction Time** (`recorded_at`) denoting system ingestion time.

* **Obsolete Fact:** Update `valid_until = NOW()` on the old edge and append a new active edge.
* **Contextual Coexistence:** Keep both edges `IN`, contextualized by environment attributes (e.g., `runtime: "v1.x"` vs `runtime: "v2.x"`).
* **Contradictory Conflict:** Isolate within an Epistemic Branch for human review.

### 3.3 Cognitive Knowledge Metabolism & Topological Decay

To prevent noise accumulation and compute unboundedness, the system fuses the **Ebbinghaus Forgetting Curve** with the **ACT-R Base-Level Activation Model** to calculate dynamic symbolic memory strength $S(v, t)$:

$$S(v, t) = e^{-\lambda \cdot (t - t_{\text{last}})} \cdot \left[1 + \alpha \cdot \log_2(1 + N_{\text{access}})\right] \cdot \left[1 + \beta \cdot \log_2(1 + \text{Deg}(v))\right]$$

* $t - t_{\text{last}}$: Elapsed duration since the last successful retrieval or reference.
* $\lambda$: Base decay rate (where axiomatic nodes set $\lambda \to 0$ for perpetual retention).
* $N_{\text{access}}$: Total retrieval access count (spacing effect).
* $\text{Deg}(v)$: Node degree centrality, granting decay immunity to topological hubs.

#### Tri-Tier Memory Scheduling

* **Active Working Memory ($S(v, t) \ge \tau_{\text{active}}$):** Full Markdown prose, source code, and graph edges remain fully indexed in fast storage.
* **Compressed Summary Layer ($\tau_{\text{archive}} \le S(v, t) < \tau_{\text{active}}$):** LLM worker threads summarize redundant derivation traces and consolidate homogeneous cards into abstract conceptual overviews.
* **Cold Archive Layer ($S(v, t) < \tau_{\text{archive}}$):** Unloaded from active FTS5 and runtime graph engines into compressed cold storage, retaining only skeleton stub pointers in the main repository.

### 3.4 Zero-Embedding Multi-Stage Retrieval & Non-Linear Ranking

```
User Query
    │
    ├──► Lexical Channel (FTS5 BM25) ──────► R_BM25
    │                                          │
    │                                          ▼
    └──► Graph Channel (PPR Diffusion) ────► R_PPR ──► Reciprocal Rank Fusion (RRF)
                                                              │
                                                              ▼
                                                     Lightweight Reranker ──► Context Assembly

```

1. **Lexical Channel:** SQLite FTS5 Okapi BM25 scores Markdown frontmatter, titles, and body content to produce candidate set $R_{\text{BM25}}$.
2. **Topological Diffusion Channel:** Extracted seed entity set $S$ initializes personalization vector $\mathbf{p}$ ($p_i = 1/\vert{}S\vert{}$ for $i \in S$, else $0$). Personalized PageRank iteratively executes across adjacency matrix $\mathbf{M}$ until convergence ($\gamma = 0.15$):

$$\mathbf{r} = (1 - \gamma) \mathbf{M} \mathbf{r} + \gamma \mathbf{p}$$



yielding structural candidate set $R_{\text{PPR}}$.
3. **Reciprocal Rank Fusion (RRF):** Reconciles disparate lexical scores and probability scales ($k = 60$):

$$\text{RRF}(d) = \sum_{m \in \{\text{BM25}, \text{PPR}\}} \frac{1}{k + r_m(d)}$$


4. **Cross-Encoder Verification:** Top $M$ RRF candidates pass to a lightweight cross-encoder or Small Language Model (SLM) for contextual pruning and final assembly.

---

## 4. Engineering Edge Cases & Pragmatic Trade-offs

### 4.1 Scaling Bounds & Non-linear Update Latencies

* **Performance Sweet Spot:** Between 1,000 and 20,000 atomic cards (~100,000 graph entities), SQLite FTS5 with 2-hop Recursive CTEs completes queries in under 5ms using dozens of megabytes of RAM.
* **Million-Scale Mitigation:** At multi-million entity scales, recursive SQL triggers combinatorial path explosion. The system mitigates this by:
* Deploying regex/dictionary rule filters on ingress to bypass LLM compilation for low-entropy text.
* Migrating from relational recursive SQL to dedicated graph engines (e.g., Kùzu) or applying the **Leiden Community Detection Algorithm** to isolate graph traversals within local partitions.



### 4.2 Knowledge Granularity: Strict Triples vs. Atomic Markdown Cards

* **Strict Triples (SPO):** Maximizes formal validation but causes severe *semantic thinning*, stripping away code context, boundary assumptions, and subtle causal qualifiers.
* **Fixed Chunking:** Preserves prose context but lacks addressable symbolic linkages.
* **The Sweet Spot: Atomic Markdown Cards:** 200–800 word scoped cards. Structured metadata and dependencies reside in the YAML frontmatter, narrative arguments and code reside in the body, and weak relational associations map via `[[wikilinks]]`. This maximizes human readability, LLM reasoning, and AST compilation performance.

### 4.3 Ingestion Rhythm: Synchronous Blocking vs. Asynchronous Batch Compilation

A fully synchronous ingestion loop causes unviable latency overhead (10–30s per ingest due to LLM extraction, directory compilation, and disambiguation). The architecture implements a **Write-Ahead Log (WAL) pattern**:

* **Synchronous Hot Path (< 5ms):** Appends raw inputs to an append-only store (`raw/` or SQLite `episodes`), immediately updating the local FTS5 index.
* **Asynchronous Cold Path:** Background workers batch-process raw events to execute structured entity extraction, frontmatter linking, directory re-compilation, and JTMS consistency checks.

### 4.4 The Multi-Dimensional Scoring Hyperplane

Retrieval quality is governed by a unified non-linear ranking hyperplane:

$$\text{FinalScore}(d, q) = w_1 \cdot \text{RRF}_{\text{Score}}(d, q) + w_2 \cdot S(d, t) + w_3 \cdot \text{ContextRelevance}(d, q)$$

Where $S(d, t)$ factors temporal activation and degree centrality, while $\text{ContextRelevance}(d, q)$ applies small-model semantic entailment verification.

### 4.5 Human-in-the-Loop (HITL) Safety Guardrails

To prevent runaway semantic drift and catastrophic overwrites, the system operates on a **Fail-Closed Governance Matrix**:

* **Low Risk:** Autonomous agent commits directly to `main` (e.g., appending isolated leaf cards, adding non-conflicting reference links).
* **Medium Risk:** Automated execution with mandatory reasoning traces recorded in the Git commit log (e.g., updating entity properties, merging verified aliases).
* **High Risk (Automated Circuit Breaker):** Changes are demoted to a Pull Request requiring human review under any of these conditions:
1. Mutations targeting foundational cards flagged as immutable global axioms (`Immutable / Canon`).
2. JTMS propositional conflicts that invert belief states for nodes within the top 5% degree centrality.
3. AST static analysis detects that a card deletion or overwrite creates more than 5 dangling downstream dependencies.



---

## 5. Deployment Architectures Across Data Scales

| Deployment Scale | Recommended Architecture | Storage Substrate | Retrieval Strategy | Governance Level |
| --- | --- | --- | --- | --- |
| **Personal / Edge**<br>

<br>*(< 5,000 cards / < 5M words)* | Pattern A (Markdown Tree) + Pattern C (Lightweight Git) | Local Filesystem + Git tracking | Agent Directory Navigation (`llms.txt`) + BM25 | Highly Autonomous; resolves edge conflicts via local user prompts |
| **Team / Domain Specific**<br>

<br>*(5,000–50,000 cards / Medium Graph)* | Pattern B (SQLite Graph) + Pattern C (CI Pipeline) | SQLite (WAL Mode + FTS5 Table) | SQLite FTS5 + 2-Hop CTE + PPR + RRF | PR Review Queue; structural modifications require maintainer sign-off |
| **Enterprise Knowledge Mesh**<br>

<br>*(> 50,000 cards / Millions of entities)* | Pattern B (Dedicated Engine) + Hybrid Compiler Mode | Embedded Graph DB (Kùzu / LatticeDB) + Object Store | Inverted Seeds + Native WCOJ Traversal + Dedicated Reranker | Multi-role enterprise governance with strict policy guardrails |

---

## 6. Architectural Conclusion

Eliminating vector databases does not regress system capabilities back to brittle keyword searches. Instead, it elevates the LLM from a **black-box geometric similarity calculator** to a high-order **Semantic Compiler and Topological Reasoning Engine**.

By orchestrating Markdown hierarchy trees, bi-temporal symbolic graphs, and Git-native review pipelines, the system establishes sub-millisecond deterministic path resolution while preserving rich contextual nuance. Augmented by Justification-Based Truth Maintenance (JTMS) and cognitive metabolic decay models, this neuro-symbolic paradigm provides a transparent, auditable, version-controlled, and self-evolving foundation for enterprise knowledge engineering.