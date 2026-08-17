# Kinetic Engine — Source Tree and Test Audit

## Audit target
`extensions/kinetic-core/kinetic-engine` on `main`.

## Purpose
This is the reconciliation pass before declaring the engine study complete. The repository source tree was enumerated directly rather than inferred from earlier notes.

## Top-level engine source inventory

The current `src/` tree contains these root files/modules:

- `ai_client.rs`
- `audio.rs`
- `bridge.rs`
- `code_exec_agent.rs`
- `embedder.rs`
- `execution_ledger.rs`
- `file_enumerator.rs`
- `file_hasher.rs`
- `gpu_detector.rs`
- `graph_retriever.rs`
- `infrastructure_master.rs`
- `main.rs`
- `mcp.rs`
- `model_capabilities.rs`
- `name_index.rs`
- `orchestrator.rs`
- `progress.rs`
- `query_engine.rs`
- `regression_guardrails.rs`
- `symbol_extractor.rs`
- `symbol_graph.rs`
- `vector_engine.rs`
- `workspace_indexer.rs`

and three nested source modules:

- `indexer/`
- `intent/`
- `pillars/`
- `tools/`

`main.rs` confirms the production module graph: the orchestration/AI/tool/pillar/intent modules are private, while the indexing/retrieval/storage modules are publicly exported to the engine crate. `regression_guardrails` is compiled only under tests. 

## Nested source inventory

### `indexer/`

Verified files:

- `mod.rs`
- `job_queue.rs`
- `query_cache.rs`
- `retrieve.rs`
- `config.rs`
- `auth.rs`
- `state.rs`
- `http.rs`
- `serve.rs`

Covered by the indexer study record and cross-referenced with MCP/Team server paths.

### `intent/`

Verified files:

- `mod.rs`
- `classifier.rs`
- `mission_builder.rs`
- `tests.rs`

The golden tests explicitly pin the core `(UserMode × TaskIntent) → ExecutionLane` contract. Examples include Code mode code changes routing to `CodeExec`, pure questions downgrading to `AskResponse`, Plan mode artifacts routing to `PlanArtifact`, ASK mode capping code changes, Vision attachment workflows routing to `VisionClone`, and Auto mode allowing code execution. This is an executable compatibility contract, not merely documentation.

### `pillars/`

Verified files:

- `mod.rs`
- `state_tracker.rs`
- `safety_gate.rs`
- `context_manager.rs`
- `memory_bridge.rs`
- `feedback_signal.rs`
- `risk_engine.rs`

All six are accounted for in the pillar study. The active file/command deterministic safety gate is distinguished from the reserved/scaffolded `risk_engine`.

### `tools/`

The current directory inventory includes:

- `mod.rs`
- `browser.rs`
- `command_risk.rs`
- `file_system.rs`
- `image_generation.rs`
- `policy.rs`
- `pulse_tools.rs`
- `sandbox.rs`
- `schemas.rs`
- `vision_assets.rs`

Some tool files are intentionally tiny stubs/placeholders. They must not be counted as implemented merely because the module exists.

## Build/runtime inventory

`Cargo.toml` establishes the engine as a Rust 2021 Tokio sidecar with:

- async runtime and JSON-RPC-style IPC;
- HTTP client/streaming;
- LanceDB + Arrow storage;
- FastEmbed;
- Tree-sitter grammars for Rust/TypeScript/JavaScript/Python/Go/Java;
- petgraph GraphRAG;
- Axum/Tower Team indexer HTTP server;
- audio capture;
- image/SVG conversion;
- optional CUDA feature;
- cross-platform Windows/Unix process priority support;
- size-oriented release profile with LTO, single codegen unit, strip, and abort-on-panic.

`main.rs` confirms two runtime modes:

```text
(no subcommand) / mcp → stdin/stdout IDE sidecar
serve → Team shared indexer HTTP API
```

Startup also applies reduced process priority, loads `.env`, initializes GPU/embedder cache state, and records the canonical indexing marker.

## Test architecture

There is no separate `tests/` directory at the engine root on the current branch. Tests are primarily colocated in source modules and the dedicated `regression_guardrails.rs` test module.

`regression_guardrails.rs` is particularly important because it pins architectural invariants rather than only ordinary function behavior. It currently guards, among other things:

- RRF `k = 60`;
- IVF-PQ threshold = 50,000 rows;
- embedder singleton shape (`OnceLock<Mutex<TextEmbedding>>`);
- parallel BM25/vector retrieval using `tokio::join!`;
- RRF fusion formula;
- bounded distinct-file counting;
- IVF threshold short-circuit;
- schema-migration warning before destructive table drops;
- canonical WorkspaceIndexer→Lance pipeline marker;
- graph ID whitelist;
- path-component exclusions instead of substring exclusions;
- schema-aware hash loading;
- no inline compaction on incremental `update_file`;
- bounded progress channel;
- file-size cap before reading file content.

This means the engine contains an explicit regression layer protecting decisions discovered during prior indexing/performance/security audits.

## Source-vs-documentation reconciliation

The source inventory agrees with the major architecture sections already documented:

```text
MCP / Team HTTP
       ↓
Orchestrator
       ├── Intent / Mission
       ├── AI Client
       ├── Tool policy / safety / Sentry
       ├── Execution ledger
       └── Context / memory / feedback pillars

Workspace Indexer
       ↓
FileHasher + SymbolExtractor + Embedder
       ↓
VectorEngine / LanceDB
       ↓
QueryEngine
       ├── BM25
       ├── Vector
       ├── RRF
       └── GraphRetriever
```

No additional root source module was found that invalidates this high-level map.

## Important implementation-state findings

The engine contains a mixture of production code and deliberately incomplete compatibility/future modules. Examples already verified include:

- `bridge.rs` — placeholder;
- `infrastructure_master.rs` — placeholder;
- `pillars::risk_engine` — reserved/scaffolded rather than the current command-risk authority;
- tiny tool modules such as `browser.rs`, `pulse_tools.rs`, and `sandbox.rs` — module presence alone does not imply functionality;
- Team `/register` — explicit future/shard registration placeholder.

These are recorded as **not implemented** rather than silently promoted into the architecture's feature inventory.

## Engine study status

### Structurally covered

- entrypoint/runtime modes;
- MCP transport and dispatch;
- Team indexer;
- indexing lifecycle;
- vector storage;
- embedding;
- hashing/incremental indexing;
- hybrid retrieval;
- GraphRAG;
- orchestration control plane;
- execution lifecycle and self-healing;
- intent/lane authority;
- mission building;
- tool policy/schema/risk model;
- native tool surface;
- safety/context/memory/feedback pillars;
- GPU/runtime support;
- execution ledger;
- regression guardrails;
- Cargo/build/runtime configuration.

### Final closure still required

The engine should not be declared complete until the following are explicitly reconciled:

1. remaining sections of the very large `orchestrator.rs` have no undocumented public execution branch;
2. all tool files listed above have been opened and their actual implementation/stub state recorded;
3. `intent/tests.rs` and any colocated tests are represented in the feature contract;
4. every pillar file has call-site evidence showing whether it is active, passive, or reserved;
5. all `mcp.rs` method names have been reconciled against TypeScript callers in `kinetic-core` during the later Core phase;
6. source documentation does not claim behavior that is only present in tests/comments;
7. the engine study master index records exact source paths for every subsystem;
8. a final call-site scan confirms no alternate indexing write path exists outside the canonical path.

## Integrity rule

This architecture repository is documentation only. No source code in `KineticIDE` is modified by this study. The source repository remains the implementation authority; this repository records the understanding of that implementation and explicitly distinguishes implemented, partial, stub, and future behavior.
