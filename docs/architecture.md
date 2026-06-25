# Architecture

> Reflects code-review-graph **v2.3.3**. Facts in this document (tool/prompt
> counts, schema version, pipeline ordering, edge kinds) were derived directly
> from the source via the project's own knowledge graph and verified against the
> code. Keep it in sync when the structure changes.

## 1. System Overview

`code-review-graph` maintains a persistent, incrementally-updated **knowledge
graph** of a codebase to make AI-assisted code review faster, cheaper (fewer
tokens), and structurally aware. It parses source with Tree-sitter, stores a
structural graph in SQLite, enriches it with derived analytics (execution flows,
communities, risk index, embeddings), and exposes everything through **30 MCP
tools + 5 MCP prompts**, a CLI, and a VS Code extension.

The package is ~29 KLOC of Python (`code_review_graph/`) plus a TypeScript VS
Code extension (`code-review-graph-vscode/`).

### Layered view

```
┌──────────────────────────────────────────────────────────────────────┐
│ Clients                                                                │
│   Claude Code / Cursor / Codex / 13+ MCP hosts   •   VS Code extension │
│   CLI (uv run code-review-graph …)               •   PostToolUse hooks │
└───────────────┬───────────────────────────────────────┬───────────────┘
                │ MCP (stdio / streamable-http)          │ read-only SQLite
                ▼                                         ▼
┌──────────────────────────────────────────┐   ┌────────────────────────┐
│ Interface layer                          │   │ VS Code extension       │
│   main.py  — FastMCP server              │   │   SqliteReader (RO)     │
│   tools/*  — 30 tools (11 modules)       │   │   tree views, webview   │
│   prompts.py — 5 prompt templates        │   │   blast-radius, search  │
└───────────────┬──────────────────────────┘   └────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────────────────┐
│ Analysis & derivation layer                                              │
│   flows.py · communities.py · changes.py · refactor.py · analysis.py     │
│   search.py · hints.py · embeddings.py · wiki.py · visualization.py      │
└───────────────┬──────────────────────────────────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Core engine                                                               │
│   parser.py (+ 4 resolvers) ──► graph.py (GraphStore) ──► postprocessing  │
│   incremental.py (git diff)      migrations.py (schema v9)                 │
└───────────────┬───────────────────────────────────────────────────────────┘
                ▼
        SQLite  .code-review-graph/graph.db   (WAL mode)
        +       embeddings DB (separate file)
```

## 2. Component Map

| Layer | Module(s) | Responsibility |
|-------|-----------|----------------|
| **Interface** | `main.py` | FastMCP server; registers 30 tools + 5 prompts; stdio or streamable-http transport; optional `--tools` allowlist and `--auto-watch`. |
| | `tools/` (11 modules) | Tool implementations (see §6). |
| | `prompts.py` | 5 guided prompt templates with a token-efficiency preamble. |
| | `cli.py` | CLI dispatch (`install`, `build`, `update`, `postprocess`, `watch`, `daemon`, `status`, `visualize`, `wiki`, `detect-changes`, `register/unregister/repos`, `serve`/`mcp`, `eval`). |
| **Parsing** | `parser.py` | Tree-sitter multi-language AST extraction → `NodeInfo` / `EdgeInfo`. |
| | `jedi_resolver.py` | Python post-parse call enrichment (resolves dropped method calls via Jedi). |
| | `spring_resolver.py` | Java Spring DI: bare method call → injected field type → concrete impl. |
| | `temporal_resolver.py` | Java Temporal workflow/activity stub call resolution. |
| | `rescript_resolver.py` | ReScript cross-module reference canonicalization. |
| | `tsconfig_resolver.py` | TypeScript path-alias (`@/…`) resolution for import edges. |
| **Storage** | `graph.py` | `GraphStore`: SQLite-backed nodes/edges, impact-radius BFS, sanitized exports. |
| | `migrations.py` | Schema migrations v1→**v9**; idempotent, version-gated. |
| | `graph_diff.py` | Snapshot diffing (added/removed nodes, edges, community changes). |
| | `memory.py` | Persistent Q&A "memory" markdown under `.code-review-graph/memory/`. |
| **Build/Update** | `incremental.py` | git/svn change detection, dependent expansion, full vs incremental build. |
| | `postprocessing.py` / `tools/build.py` | Post-processing pipeline orchestration + summary precomputation. |
| | `enrich.py` | PostToolUse/PreToolUse hook: emits graph context for Grep/Read/Bash. |
| | `daemon.py` / `daemon_cli.py` | Multi-repo watch daemon (spawns one watcher per repo). |
| **Analysis** | `flows.py` | Entry-point detection, flow tracing, criticality scoring. |
| | `communities.py` | Leiden (igraph) or file-based community detection + cohesion + coupling warnings. |
| | `changes.py` | Diff→node mapping, per-node risk scoring (`detect-changes`). |
| | `refactor.py` | Rename preview, dead-code detection, suggestions, `apply_refactor`. |
| | `analysis.py` | Hub (degree), bridge (betweenness), knowledge gaps, surprising connections. |
| | `search.py` | FTS5 + vector hybrid search via Reciprocal Rank Fusion. |
| | `hints.py` | Session-state intent inference + `next_tool_suggestions`. |
| **Periphery** | `embeddings.py` | 4 embedding providers (Local, OpenAI-compatible, Gemini, MiniMax). |
| | `visualization.py` | Self-contained D3.js HTML graph (SRI-pinned CDN). |
| | `exports.py` | GraphML / Cypher / Obsidian / SVG export. |
| | `wiki.py` | Markdown wiki generation from community structure. |
| | `skills.py` | Multi-platform MCP/skill/hook installer (13+ hosts). |
| | `registry.py` | Multi-repo registry + thread-safe connection model. |
| | `eval/`, `token_benchmark.py` | Evaluation/benchmark framework. |

## 3. Data Model (SQLite, schema v9)

Nodes and edges live in the primary DB (`.code-review-graph/graph.db`, WAL mode);
embeddings live in a **separate** DB file. The schema is established by
`_SCHEMA_SQL` in `graph.py` and evolved by `migrations.py`.

### Core tables

**`nodes`** — one row per structural entity
`id`, `kind` (File · Class · Function · Type · Test), `name`,
`qualified_name` (UNIQUE), `file_path`, `line_start`, `line_end`, `language`,
`parent_name`, `params`, `return_type`, `modifiers`, `is_test`, `file_hash`,
`signature` *(v2)*, `community_id` *(v4)*, `extra` (JSON), `updated_at`.

**`edges`** — one row per relationship
`id`, `kind`, `source_qualified`, `target_qualified`, `file_path`, `line`,
`confidence` *(v9, default 1.0)*, `confidence_tier` *(v9, default `EXTRACTED`)*,
`extra` (JSON), `updated_at`.

**`metadata`** — key/value (`last_updated`, `build_type`, `schema_version`, …).

### Derived/analytics tables

| Table | Added | Purpose |
|-------|-------|---------|
| `flows`, `flow_memberships` | v3 | Execution flows: entry point, depth, criticality, ordered path. |
| `communities` | v4 | Detected clusters: name, level, cohesion, size, dominant language. |
| `nodes_fts` (FTS5) | v5 | Full-text search over name/qualified_name/file_path/signature. |
| `community_summaries`, `flow_snapshots`, `risk_index` | v6 | Denormalized summaries for cheap reads (top symbols, critical paths, per-node risk). |

### Migration history

| Ver | Change |
|-----|--------|
| v1 | Base schema (`nodes`, `edges`, `metadata`). |
| v2 | `nodes.signature`. |
| v3 | `flows`, `flow_memberships`. |
| v4 | `communities` + `nodes.community_id`. |
| v5 | `nodes_fts` FTS5 virtual table. |
| v6 | `community_summaries`, `flow_snapshots`, `risk_index`. |
| v7 | Compound edge indexes `(target,kind)` / `(source,kind)`. |
| v8 | Composite covering index on edges. |
| v9 | `edges.confidence` + `edges.confidence_tier`. |

`LATEST_VERSION = max(MIGRATIONS.keys())`; `run_migrations()` is a no-op when the
DB is already current.

### Qualified names

- **File**: the file path (e.g. `code_review_graph/parser.py`).
- **Top-level symbol**: `file_path::name` (e.g. `…/parser.py::CodeParser`).
- **Method / nested**: `file_path::Parent.name` (e.g. `…/graph.py::GraphStore.upsert_edge`).

### Edge kinds

`CALLS`, `IMPORTS_FROM`, `INHERITS`, `IMPLEMENTS`, `CONTAINS`, `TESTED_BY`,
`DEPENDS_ON`, `REFERENCES` (plus enrichment-specific markers such as `INJECTS` /
Temporal stub roles carried in `extra`). Edges default to `confidence_tier =
EXTRACTED`; the `RESOLVED` tier is reserved (resolvers currently rewrite
`target_qualified` rather than relabel the tier).

## 4. Parsing Pipeline

`CodeParser.parse_file(path)` reads bytes **once**, hashes them, then calls
`parse_bytes(path, source)` (TOCTOU-safe). `parse_bytes` dispatches by detected
language:

1. **Language detection** — extension map (`EXTENSION_TO_LANGUAGE`, ~35 entries)
   or shebang interpreter. Covers ~20 base languages plus TSX/JSX, Luau, etc.
   Special formats get dedicated handlers: **Vue SFC**, **Svelte SFC**, **Jupyter
   / Databricks notebooks**, **ReScript** (regex-based, no bundled grammar),
   **SQL** (regex on `CREATE TABLE/FUNCTION/PROCEDURE`).
2. **Tree-sitter walk** — recursive AST traversal pattern-matches node types
   against language-specific maps (class/function/import types) rather than
   tree-sitter queries, for resilience across grammar versions.
3. **Extraction** — produces `NodeInfo` (kind, name, file_path, line span,
   language, parent, params, return_type, modifiers, is_test, extra) and
   `EdgeInfo` (kind, source, target, file_path, line, extra).

### Post-parse resolvers (enrichment)

Run after the raw parse to recover edges Tree-sitter alone cannot resolve.
They rewrite/insert edges in-place:

| Resolver | Language | What it fixes |
|----------|----------|---------------|
| ReScript | ReScript | Bare `Module.fn` targets → canonical `file::fn`; handles `open`/`include`, JSX components. |
| Spring DI | Java | Bare method call on injected field → concrete impl via `INJECTS` field map + `INHERITS`. |
| Temporal | Java | Receiver var → Temporal stub type → concrete workflow/activity impl. |
| Jedi | Python | Re-walks ASTs, uses `jedi.Script.goto()` to add dropped `CALLS` edges. |

## 5. Build & Update Flow

### Full build (`code-review-graph build`)

1. **Collect** tracked files (`git ls-files`) minus `.code-review-graphignore` /
   gitignored.
2. **Parse** all files in parallel (process pool; falls back to threads on
   Windows MCP to avoid `ProcessPoolExecutor` deadlocks).
3. **Store** nodes/edges per file in `BEGIN IMMEDIATE` transactions; stale rows
   purged first.
4. **Resolvers** (ReScript → Spring → Temporal; Jedi separately for Python).
5. **Post-process** (level-gated, see below).

### Incremental update (`code-review-graph update`)

1. `get_changed_files(base="HEAD~1")` via `git diff --name-only` (svn supported).
2. `find_dependents()` — BFS up to 2 hops over import/call edges (≤500 files).
3. Union changed + dependents; drop ignored/deleted files.
4. **Hash-skip** files whose SHA-256 is unchanged.
5. Parse + store the survivors; re-run resolvers only for affected languages.
6. Incremental post-processing.

### Post-processing levels (`postprocess=full|minimal|none`)

| Stage | `none` | `minimal` | `full` |
|-------|:--:|:--:|:--:|
| Signatures (`def name(params)->ret`) | – | ✓ | ✓ |
| FTS5 index rebuild | – | ✓ | ✓ |
| Flow detection (`trace_flows`) | – | – | ✓ |
| Community detection | – | – | ✓ |
| Summaries (`_compute_summaries`) | – | – | ✓ |

`_compute_summaries` precomputes `community_summaries` (top-5 symbols by edge
count), `flow_snapshots` (denormalized critical paths), and `risk_index`
(caller-count + test-coverage + security-keyword scoring) using batched aggregate
queries, all under explicit transactions.

### Hook integration (`enrich.py`)

A PreToolUse hook intercepts `Grep`/`Read`/`Bash` and emits graph context as
`hookSpecificOutput.additionalContext` — the matched symbols with their callers,
callees, flows, and tests. It is a silent no-op when the graph is missing or
yields no matches. A PostToolUse hook triggers incremental updates after edits.

### Watch daemon (`daemon.py`)

`WatchDaemon` spawns one `code-review-graph watch` child per registered repo,
reads desired state from `~/.code-review-graph/watch.toml`, persists child PIDs
to `daemon-state.json`, health-checks every 30 s, and reconciles config changes
live under a lock.

## 6. MCP Interface (30 tools, 5 prompts)

The FastMCP server (`main.py`) resolves a `GraphStore` per call via
`_get_store(repo_root)` → `_validate_repo_root()` (requires `.git/` or
`.code-review-graph/`, blocking path traversal) → `get_db_path()`. Multi-repo
calls go through `registry.py`.

| Module | Tools |
|--------|-------|
| `tools/build.py` | `build_or_update_graph`, `run_postprocess` |
| `tools/query.py` | `get_impact_radius`, `query_graph`, `semantic_search_nodes`, `list_graph_stats`, `find_large_functions`, `traverse_graph` |
| `tools/context.py` | `get_minimal_context` |
| `tools/review.py` | `get_review_context`, `detect_changes`, `get_affected_flows` |
| `tools/flows_tools.py` | `list_flows`, `get_flow` |
| `tools/community_tools.py` | `list_communities`, `get_community`, `get_architecture_overview` |
| `tools/analysis_tools.py` | `get_hub_nodes`, `get_bridge_nodes`, `get_knowledge_gaps`, `get_surprising_connections`, `get_suggested_questions` |
| `tools/refactor_tools.py` | `refactor`, `apply_refactor` |
| `tools/docs.py` | `embed_graph`, `get_docs_section`, `generate_wiki`, `get_wiki_page` |
| `tools/registry_tools.py` | `list_repos`, `cross_repo_search` |

**Prompts**: `review_changes`, `architecture_map`, `debug_issue`,
`onboard_developer`, `pre_merge_check` — each prefixed with a token-efficiency
preamble.

### Token-efficiency contract

- `get_minimal_context` is the **~100-token entry point**: stats, risk, top
  communities/flows, and `next_tool_suggestions`.
- `detail_level="minimal"` (vs `standard`) drops member lists and aggregates
  per-pair edges (the architecture overview reports ~100% / ~140 KB token savings
  on this very repo).
- Every response carries `next_tool_suggestions` (capped at 3); key entities
  capped at 10, communities at 5. `hints.py` infers intent from recent calls to
  pick suggestions.

## 7. Analysis Algorithms

| Feature | Module | Core idea |
|---------|--------|-----------|
| **Flows** | `flows.py` | Detect entry points (decorators, `main`/`test_*`/`handle_*`, no-caller funcs), BFS-trace `CALLS` chains, score criticality = weighted sum of file-spread (0.30), external calls (0.20), security sensitivity (0.25), test-coverage gap (0.15), depth (0.10). |
| **Communities** | `communities.py` | Leiden via igraph (resolution `max(0.05, 1/log10 n)`) or directory-depth fallback; cohesion = internal/(internal+external) weighted edges; coupling warning when a non-test pair exceeds 10 cross edges. |
| **Change risk** | `changes.py` | Map diff line ranges → overlapping nodes; risk = flow participation (≤0.25) + cross-community callers (≤0.15) + test-coverage gap (≤0.30) + security keyword (+0.20) + caller count (≤0.10). |
| **Hubs / bridges** | `analysis.py` | Hubs by total degree; bridges by NetworkX betweenness (k=500 sampling above 5000 nodes). |
| **Surprising edges** | `analysis.py` | Composite: cross-community +0.3, cross-language +0.2, peripheral→hub +0.2, cross-test-boundary +0.15, unusual kind +0.15. |
| **Dead code** | `refactor.py` | Multi-pass filter excluding entry points, tests, dunders, constructors, framework-managed classes; validates bare-name calls and polymorphic dispatch before flagging. |
| **Hybrid search** | `search.py` | FTS5 (BM25) ⊕ vector via Reciprocal Rank Fusion (k=60), with PascalCase→Class, snake_case→Function, dotted-path and context-file boosts. |

### Impact-radius BFS (`graph.py`)

Seeds = all nodes in the changed files. Default engine is a **SQL recursive CTE**
(`BFS_ENGINE=sql`) traversing forward and backward edges up to
`MAX_IMPACT_DEPTH` (default 2), capped at `MAX_IMPACT_NODES` (default 500); a
legacy NetworkX engine is selectable. Returns changed nodes, impacted nodes,
impacted files, edges, and a `truncated` flag. The bidirectional walk captures
both downstream effects (callers of changed code) and upstream context
(dependencies of changed code).

## 8. Embeddings & Search

`embeddings.py` defines an `EmbeddingProvider` abstraction with four backends:
**Local** sentence-transformers (`all-MiniLM-L6-v2`, default, offline),
**OpenAI-compatible** (real OpenAI, Azure, LiteLLM/vLLM/Ollama gateways — the
provider name embeds the hostname to prevent silent vector-space mismatch),
**Google Gemini** (task-aware document vs query), and **MiniMax** (1536-dim).
Vectors are packed as binary blobs in a dedicated DB; the stored provider name
gates re-embedding on change. Cloud egress prints a stderr warning unless
`CRG_ACCEPT_CLOUD_EMBEDDINGS=1`. Search blends FTS5 and cosine similarity (§7).

## 9. Visualization & Export

`visualization.py` emits a self-contained D3.js v7 force-directed HTML graph in
three modes — `full`, `community` (super-nodes with drill-down), `file`. Security
controls: **SRI-pinned** D3 CDN script and `json.dumps` payloads with `</`
escaped to prevent premature `</script>` termination. `exports.py` additionally
produces GraphML (Gephi/yEd/Cytoscape), Cypher (Neo4j), Obsidian, and SVG.
`wiki.py` renders one Markdown page per community (overview, members, flows,
cross-community dependencies).

## 10. VS Code Extension

`code-review-graph-vscode/` (TypeScript) reads `.code-review-graph/graph.db`
**read-only** via `SqliteReader` (better-sqlite3). It provides tree views
(file → symbol → edge, blast-radius, stats), cursor-aware blast-radius
(`features/blastRadius.ts` resolves the innermost node at the cursor line),
semantic/keyword search, review hints, SCM decorations, and a D3 webview
(`views/graphWebview.ts`). Commands: build/update graph, show blast radius,
search nodes, review code.

## 11. Security Invariants

- **Path traversal**: `_validate_repo_root()` requires a `.git/` or
  `.code-review-graph/` marker before opening any DB.
- **Prompt injection**: `_sanitize_name()` (in `graph.py`) strips ASCII control
  chars (0x00–0x1F except tab/newline) and caps names at 256 chars on every
  node/edge export, so adversarial identifiers in source cannot steer the agent.
- **SQL injection**: all queries use `?` placeholders; no f-string values.
- **No dangerous primitives**: no `eval`/`exec`/`pickle`/`yaml.unsafe_load`, no
  `shell=True`.
- **Output safety**: visualization escapes HTML and pins D3 via SRI.
- **Secrets**: embedding API keys come only from environment variables; cloud
  egress is opt-in.
- **Concurrency**: `GraphStore` uses `check_same_thread=False`, WAL mode, a 5 s
  busy timeout, and a `threading.Lock` guarding the NetworkX cache.

## 12. Observed Structure (self-analysis)

Running the graph on its own source (171 files, ~3 K nodes, ~21.5 K edges, 16
communities, 192 flows) surfaces the expected shape:

- **Hub nodes** (highest degree): `cli.py::main`, `parser.py::{NodeInfo,
  CodeParser, EdgeInfo}`, `extension.ts::registerCommands`, `graph.py::GraphStore`,
  `refactor.py::find_dead_code`. These are the highest-blast-radius edit targets.
- **Bridge nodes** (betweenness chokepoints): `CodeParser`, `GraphStore`,
  `find_dead_code`, plus `WatchDaemon` and the v2 integration test.
- **Highest coupling**: `code-review-graph-tool` ↔ `tests-finds` (1143 edges —
  expected test-to-impl coupling, not an architectural smell).
- **Large functions** worth decomposition attention: `cli.py::main` (737 lines),
  `extension.ts::registerCommands` (685), `parser.py::_parse_rescript` (377),
  `refactor.py::find_dead_code` (328).

These are guidance signals, not defects — the test-heavy coupling and large CLI
dispatcher are intentional for a tool of this kind.
