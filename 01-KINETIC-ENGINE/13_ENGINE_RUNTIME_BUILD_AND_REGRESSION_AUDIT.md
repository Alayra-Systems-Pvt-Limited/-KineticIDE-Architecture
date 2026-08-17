# Kinetic Engine — Runtime, Build, and Regression Audit

## Status
**NEAR-COMPLETE, final closure pending repository-wide tool dispatch and TypeScript boundary verification.**

## 1. Process entrypoint

`src/main.rs` establishes two runtime modes: default `mcp` sidecar and explicit `serve` Team indexer HTTP mode.

Startup sequence:

```text
process priority adjustment
        ↓
.env load
        ↓
CLI parse
        ↓
GPU + embedder boot/cache detection
        ↓
MCP sidecar OR Team HTTP server
```

The engine intentionally lowers process priority (`nice(10)` on Linux/macOS; `BELOW_NORMAL_PRIORITY_CLASS` on Windows) because it is an IDE sidecar and should avoid starving the editor.

## 2. GPU/embedder boot contract

`KINETIC_GLOBAL_STORAGE` controls the global model cache location. If unset, FastEmbed falls back to `.fastembed_cache/` in the current working directory and offline installer pre-staging is unavailable.

GPU detection runs before the server starts. If CUDA is active and the expected DLL exists, the process sets `ORT_DYLIB_PATH` for ONNX Runtime. CUDA is optional and CPU fallback remains available.

## 3. Canonical indexing ownership

The boot marker explicitly states:

```text
WorkspaceIndexer → Lance
FileEnumerator → Vault-only file enumeration
```

`FileEnumerator` does not embed/vectorize/write LanceDB. It exists for non-vector consumers, currently the Vault secret scanner. This resolves the historical naming collision between an older `Indexer` and the actual vector indexer.

`FileEnumerator` has its own ignore policy and caps enumeration at 200,000 files / one hour. Its policy intentionally remains separate from the vector indexer's policy.

## 4. Dependency/build architecture

`kinetic-engine/Cargo.toml` confirms a native Rust sidecar using Tokio, Serde, Anyhow, Reqwest/rustls, FastEmbed, LanceDB, Arrow, tree-sitter grammars, petgraph, Axum/Tower, image/resvg/usvg/tiny-skia, git2, regex/globset, sysinfo, and optional CUDA support.

## 5. Release build strategy

The release profile uses:

```text
opt-level = "z"
lto = true
codegen-units = 1
strip = true
panic = "abort"
```

This prioritizes binary size and whole-program optimization.

## 6. Regression guardrails

`regression_guardrails.rs` encodes previous indexing audit findings as structural tests. Confirmed pinned invariants include:

- RRF `k = 60`;
- IVF-PQ threshold = 50,000 rows;
- singleton `OnceLock<Mutex<TextEmbedding>>` embedder;
- BM25 + vector through `tokio::join!`;
- RRF merge formula;
- bounded distinct-file count short-circuit;
- IVF threshold short-circuit;
- schema migration emits `SchemaMigration` before destructive table drop;
- canonical pipeline marker in `main.rs`;
- strict graph ID whitelist;
- path-component walker matching rather than substring matching;
- schema-aware `FileHasher::load_for_schema`;
- incremental `update_file` marks tables dirty rather than compacting synchronously;
- bounded progress channel;
- file-size caps before large reads.

These tests intentionally protect performance, correctness, and safety fixes from future regressions.

## 7. Engine maturity assessment

Verified implementations now include process/runtime boot, MCP sidecar, Team indexer, workspace/incremental indexing, LanceDB, FastEmbed lifecycle, hybrid BM25/vector retrieval, RRF, GraphRAG, symbol extraction/graph, revision/hash tracking, query cache, job coalescing, intent classification, lane ceilings, catalog-driven schemas, file/process tools, safety gates, command-risk classification, permission RPC, execution ledger, progress/cancellation, memory/feedback persistence, image/vision utilities, and regression guardrails.

## 8. Remaining closure items

The engine should not be declared 100% complete until explicitly verified:

1. Every one of the 20 catalog entries maps to its expected runtime executor or intentional bridge path.
2. Catalog timeout values are enforced at the dispatch boundary.
3. The complete `McpBridgeClient` lifecycle is reconciled with TypeScript `mcpBridge.ts`.
4. Rust NDJSON ownership is reconciled with the TypeScript stream contract.
5. All remaining orchestrator/`execute_plan` branches are accounted for.
6. Current Cargo tests/CI and feature builds are checked.
7. Historical audit documents are compared with current source so stale roadmap claims are not mistaken for active behavior.

Until those checks are complete, status remains **NEAR-COMPLETE**, not COMPLETE.
