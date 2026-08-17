# Kinetic Engine — Canonical Index Build, Retrieval, Embedding and Graph Storage

## Study status
**PARTIAL / ACTIVE.** This document records the implementation relationships verified so far. `workspace_indexer.rs`, `query_engine.rs`, `vector_engine.rs`, `embedder.rs`, and `graph_retriever.rs` are large modules and their remaining helper paths still need line-level completion before this subsystem is declared complete.

## 1. Canonical write invariant

`workspace_indexer.rs` explicitly defines itself as the **single canonical pipeline for all writes to the vector index**. The intended invariant is:

```text
MCP index_workspace ───────┐
MCP update_file_index ─────┼→ WorkspaceIndexer → VectorEngine/LanceDB
MCP delete_file_index ────┘
```

`enumerate_workspace_files` is deliberately outside this write path: it returns paths for the Vault Scanner and does not touch LanceDB.

This is an important architectural invariant because introducing a second direct LanceDB write path would create dual-write/ghost-row risks and undermine index auditability.

## 2. WorkspaceIndexer responsibilities

`WorkspaceIndexer` owns the end-to-end indexing pipeline and integrates:

- workspace walking (`ignore::WalkBuilder`);
- path normalization;
- binary detection;
- file-size protection;
- workspace file-count protection;
- persisted file hashes;
- Git changed-file fast path;
- incremental updates/deletions;
- SimHash-based deduplication;
- embedding;
- symbol extraction;
- symbol graph edge generation;
- name-index updates;
- progress events;
- LanceDB batch insertion.

The current implementation is intentionally defensive against pathological repositories.

### Safety budgets verified

- `EMBED_BATCH_SIZE = 256`
- `FILE_SIZE_CAP_BYTES = 10,000,000`
- default workspace file cap = `500,000`
- file-cap override: `KINETIC_INDEXER_FILE_CAP`
- process-lifetime counters expose oversize-file and file-cap events
- `LAST_SUCCESSFUL_BUILD_AT` records successful end-to-end build time

The comments explain that the 256 embedding batch aligns with fastembed's internal preferred batch for the selected model, while the 10 MB cap prevents a huge generated artifact from causing an early OOM.

## 3. Workspace normalization

`canonical_workspace_norm()` canonicalizes the workspace root with `dunce::canonicalize`, then applies the engine's file-path normalization and removes a trailing slash.

`workspace_relative()` then strips that canonical root from normalized walker paths. The result is the workspace-relative key used for exact Git changed-file matching.

This prevents absolute-path formatting differences from breaking incremental Git matching.

## 4. File admission

`is_likely_binary_bytes()` examines up to the first 8 KB and treats a NUL byte as a strong binary signal. Unreadable files are intended to be skipped rather than embedded as garbage. Empty files are treated as text.

`exceeds_file_size_cap()` performs a metadata-only size check before `fs::read`. Oversized files are counted in `OVERSIZE_FILES_SKIPPED_TOTAL` and skipped.

The walker also has a hard file-count ceiling. `KINETIC_INDEXER_FILE_CAP` can override the default when the operator intentionally supports a larger workspace.

## 5. Incremental build model

`build_index_with_options()` is the main build entry point. It:

1. emits `IndexStarted`;
2. sets job phase to `scanning`;
3. initializes/open the LanceDB code table;
4. initializes the singleton embedder;
5. loads schema-aware persisted file hashes;
6. compares the previous indexed Git commit to current Git state;
7. can fast-path when Git reports no commits since the last index;
8. otherwise proceeds through the workspace scan and incremental indexing pipeline.

The no-change fast path still emits `IndexCompleted { success: true }` and updates `LAST_SUCCESSFUL_BUILD_AT`, meaning a successful verification with zero changed files is considered a successful index run.

`triggered_by_idle` is propagated into the progress event so the dashboard can distinguish an idle/background trigger from a direct build request without changing the actual indexing algorithm.

## 6. Embedding architecture

`embedder.rs` provides a process-global `OnceLock<Mutex<TextEmbedding>>` singleton.

Current model contract:

```text
Model: JinaEmbeddingsV2BaseCode
Dimension: 768
Maximum input length: 8192 tokens
```

The same constants are intended to be the single source of truth across CPU/CUDA initialization and vector schema expectations.

The mutex is deliberate: fastembed's `TextEmbedding` is not treated as Sync-safe, so concurrent calls serialize embedding execution.

### CPU path

The default CPU factory loads the Jina code embedding model. Model files are cached under:

```text
{KINETIC_GLOBAL_STORAGE}/models/
```

when `KINETIC_GLOBAL_STORAGE` is available; otherwise fastembed's default cache is used.

The cache location is also an installer integration point: pre-populating it can make first-run operation offline after installation.

### CUDA path

CUDA is feature-gated. With the feature disabled, the CUDA factory is compiled out and returns `None`.

When compiled and `ORT_DYLIB_PATH` is present, the engine attempts CUDA initialization and falls back to CPU on any failure. The resolved provider is exposed through `embedder_status()`.

Therefore GPU acceleration is designed as an opportunistic execution-provider choice rather than a separate indexing implementation.

## 7. VectorEngine storage

`VectorEngine` wraps a LanceDB `Connection`.

The database is local to the user's global Kinetic storage area rather than the project workspace:

```text
{global_storage}/vector_store/vectors/{normalized_workspace_name}
```

An empty global-storage value falls back to a `.kinetic_ide_core` directory under the user's home directory.

This makes the index persistent across IDE sessions while avoiding pollution of the project repository.

## 8. Workspace vector schema

Current `SCHEMA_VERSION = 4`.

The primary `workspace_index` schema contains:

- `id` — non-null string
- `content` — non-null source chunk/symbol body
- `vector` — non-null 768-dimensional Float32 vector
- `timestamp` — non-null Int64
- `schema_version` — nullable Int32
- `symbol_name` — nullable
- `symbol_kind` — nullable
- `language` — nullable
- `start_line` — nullable Int32
- `end_line` — nullable Int32

The vector dimension changed from 384 to 768 in the Jina migration. The symbol metadata was introduced before that migration.

## 9. Schema migration behavior

`init_code_table()` detects whether an existing table has the expected version fields, symbol fields, and vector dimension.

The current table is accepted only when the schema contains the expected v4 structure and 768-dimensional vector.

Older structures are treated as destructive migrations: the table is counted, a `SchemaMigration` progress event is emitted, the old table is dropped, and a fresh table is created.

This is an important operational property: **the vector index is disposable/rebuildable derived state**, while the source workspace remains authoritative.

The FTS index for `content` is created when the workspace table is newly created, enabling the BM25 branch of hybrid retrieval.

## 10. Symbol graph storage

`VectorEngine` maintains a separate `symbol_graph` table with `GRAPH_SCHEMA_VERSION = 1`.

Fields:

- `from_id`
- `to_id`
- `edge_type`
- `weight`
- `schema_version`

The table represents relationships such as calls/imports/inheritance/usage/definition relationships. Scalar indexes on `from_id` and `to_id` support graph expansion.

Graph schema migration is independent from the vector-index schema.

## 11. Hybrid retrieval

`query_engine.rs` implements hybrid retrieval against the LanceDB `workspace_index`.

The high-level flow is:

```text
query text
   ↓
singleton embedder → query vector
   ↓
 ┌───────────────────────┐
 │ BM25 lexical search   │
 │ Vector semantic search│  ← tokio::join!, parallel
 └───────────────────────┘
   ↓
Reciprocal Rank Fusion
   ↓
GraphRAG depth-1 expansion
   ↓
Path filtering
   ↓
Top-K truncation
   ↓
RetrievalChunk[]
```

`RRF_K = 60.0` is pinned as a public constant and protected by regression tests. Each result contributes:

```text
1 / (60 + rank)
```

from each retrieval fork.

The two search forks run concurrently, so combined latency targets the slower branch rather than the sum of both branches.

## 12. Degraded hybrid search

BM25 and vector retrieval are independently tracked as `bm25_ok` and `vector_ok`.

A failure in one branch is **non-fatal**: the successful branch's results can still be merged. This means retrieval degrades rather than completely failing when one search mechanism is unhealthy.

If the workspace index does not exist, the result explicitly reports:

- empty chunks;
- `Workspace is not indexed yet.`;
- graph status `Missing`;
- both retrieval health flags false.

This prevents an unindexed workspace from being reported as healthy search.

## 13. GraphRAG query-time expansion

`graph_retriever.rs` performs depth-1 graph expansion after RRF.

Current query-time semantics:

- seed count from query engine: 10;
- neighbors per seed in that path: 3;
- default graph module constant: 5;
- proximity decay: `0.9`;
- neighbor score = parent score × proximity decay × edge weight.

Both outgoing and incoming graph edges are queried in parallel. A `petgraph::DiGraph` is then built in memory to perform the one-hop traversal and deduplicate relationships.

Graph status is explicit:

- `missing` — graph table absent;
- `empty` — graph exists but produced no neighbor;
- `ready` — neighbors returned;
- `no_seeds` — upstream RRF produced no seeds, so graph health is unknown from that query.

This is a deliberate observability distinction rather than treating every empty result as the same failure state.

## 14. Graph query security/recall safeguards

Because the current LanceDB Rust SDK path constructs `IN (...)` filters as strings, graph IDs are escaped and additionally validated by a strict whitelist before insertion.

Allowed IDs cover realistic path/symbol syntax (`A-Z`, digits, `.`, `_`, `/`, `:`, `#`, `@`, `$`, `-`) with a 512-character ceiling.

Graph telemetry tracks:

- neighbor IDs requested but not found in `workspace_index`;
- graph edges rejected because their IDs fail validation.

This makes stale graph references and malformed extractor output observable instead of silently corrupting query behavior.

## 15. End-to-end retrieval contract

The combined architecture now verified is:

```text
MCP / Team HTTP
      ↓
Indexer retrieve facade
      ↓
query_cache (60 s)
      ↓
QueryEngine
      ├── embed query via global Jina singleton
      ├── BM25 ───────┐
      └── Vector ─────┤ parallel
                      ↓
                   RRF(K=60)
                      ↓
                 GraphRAG 1-hop
                      ↓
                path filtering
                      ↓
                   Top-K
                      ↓
             RetrievalChunk[]
```

The durable source of truth is still the workspace source tree; LanceDB is a derived local index that can be rebuilt when schema/model/index state changes.

## 16. Remaining study work

Before declaring this portion complete, inspect the remaining implementations/call sites for:

- the full `WorkspaceIndexer` scan → chunk/symbol → embed → insert/delete lifecycle;
- `symbol_extractor.rs`;
- `symbol_graph.rs`;
- `name_index.rs`;
- `file_hasher.rs` and SimHash persistence;
- all remaining `VectorEngine` mutation/optimization functions;
- all remaining `QueryEngine` helpers and tests;
- graph content fetch and SQL escaping implementation;
- progress/health integration;
- MCP callers and tests that exercise these paths.

No component should be marked complete merely because its primary file has been inspected; its callers and downstream effects must also be traced.
