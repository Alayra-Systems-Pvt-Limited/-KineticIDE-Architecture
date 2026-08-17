# Kinetic Engine Study

## Scope
Complete implementation study of:
`KineticIDE/extensions/kinetic-core/kinetic-engine`

The source is not modified. This directory records observations against the source repository and its exact revision.

## Source revision at initialization
- Branch: `main`
- Commit observed: `bd8a1b378d7142e0e6e97d6619c3b4ac315bc29c`
- Crate: `kinetic-engine`
- Version: `0.1.0`
- Rust edition: `2021`

## Completion criteria
This phase is complete only when:
1. Every file and nested directory under `kinetic-engine` has been enumerated.
2. Every source/config/build/documentation/log/artifact file relevant to runtime/build behavior has been inspected or explicitly marked non-source/generated/binary and recorded.
3. Every Rust module has been read in full, including tests where present.
4. Important call paths are traced across module boundaries.
5. Features are mapped to concrete implementation paths.
6. Security, safety, persistence, networking, process boundaries, and failure behavior are documented.
7. Any uncertainty or inaccessible file is explicitly recorded as `BLOCKED` rather than guessed.

## Initial crate map
`main.rs` declares these engine modules:

- execution_ledger
- mcp
- model_capabilities
- code_exec_agent
- orchestrator
- file_enumerator
- pillars
- intent
- tools
- ai_client
- audio
- vector_engine
- workspace_indexer
- symbol_extractor
- symbol_graph
- graph_retriever
- name_index
- progress
- query_engine
- embedder
- file_hasher
- gpu_detector
- indexer

`IndexerMode` is re-exported from `indexer`.

## Entry-point behavior already verified
`main()`:
1. lowers process priority (`BELOW_NORMAL_PRIORITY_CLASS` on Windows; `nice(10)` on Linux/macOS);
2. loads `.env` through `dotenvy`;
3. parses CLI/subcommands;
4. initializes GPU/embedder cache state;
5. defaults to MCP sidecar mode when no subcommand is provided;
6. supports `serve` mode for the team shared indexer.

`init_gpu_and_embedder_cache()` checks global model storage, detects GPU state, configures `ORT_DYLIB_PATH` when CUDA is active and the expected DLL exists, records boot GPU information, and logs the canonical vector-index pipeline.

`run_mcp_sidecar()` constructs `McpServer` and passes a mutable `Orchestrator` to its listener.

`run_team_serve()` builds `indexer::config::ServeConfig` from CLI/env arguments and delegates to `indexer::serve::run`.

## Dependency-level architecture signals
From `Cargo.toml`, the engine uses:
- Tokio, futures/futures-util for async/concurrency.
- Serde/serde_json and anyhow for data/error handling.
- Reqwest + rustls-backed default TLS for provider/HTTP communication.
- LanceDB + Arrow for vector persistence.
- FastEmbed for local embeddings.
- Tree-sitter grammars for Rust/TypeScript/JavaScript/Python/Go/Java symbol extraction.
- Petgraph for in-memory graph expansion/GraphRAG.
- Axum/Tower/Tower-HTTP for HTTP serving.
- Git2 for Git integration.
- Ignore/globset/regex/dunce/sha2/uuid for workspace traversal, matching, hashing, and IDs.
- CPAL/Hound/base64 for audio capture/encoding.
- Image/resvg/usvg/tiny-skia/urlencoding for vision/image processing.
- Optional `cuda` feature through FastEmbed.
- Size-focused release profile: opt-level z, LTO, one codegen unit, strip, panic abort.

These are **dependency signals, not final feature conclusions**. Each must be validated against the implementation during the file study.

## Progress
- [x] Study ledger initialized
- [x] Crate manifest read
- [x] Entry point read
- [ ] Complete tree inventory
- [ ] Complete module study
- [ ] Nested subsystem study
- [ ] Tests/guardrails study
- [ ] Cross-module call graph
- [ ] Feature map
- [ ] Security/safety audit
- [ ] End-to-end execution flows
- [ ] Final engine completion review

## Next action
Continue with complete tree enumeration and then inspect each subsystem in dependency/entry-point order. Do not proceed to Kinetic Core until this checklist is complete.
