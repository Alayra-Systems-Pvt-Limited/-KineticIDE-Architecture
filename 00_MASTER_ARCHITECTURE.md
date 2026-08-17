# KineticIDE Architecture Study — Master Ledger

## Purpose
This repository is the durable technical knowledge base for studying the existing KineticIDE implementation. It must preserve enough implementation-level context that the study can be resumed after long gaps in conversation.

## Source of truth
- Source repository: `Alayra-Systems-Pvt-Limited/KineticIDE`
- Source branch studied: `main`
- Source commit observed at study start: `bd8a1b378d7142e0e6e97d6619c3b4ac315bc29c`
- This architecture repository is separate from the product repository.

## Non-modification rule
The KineticIDE source repository is treated as **read-only for this study**. No source file, configuration, documentation, generated artifact, or other byte is to be modified, deleted, renamed, or reformatted as part of the architecture study.

## Study order
1. `extensions/kinetic-core/kinetic-engine` — COMPLETE FILE-BY-FILE STUDY FIRST
2. `extensions/kinetic-core` — after engine is understood
3. `extensions/kinetic-ui` — after core is understood
4. Cross-layer end-to-end audit

## Study method
For every source file we record:
- path and source revision
- purpose and responsibility
- public API/types/functions
- dependencies and callers/callees where traceable
- state ownership and lifecycle
- I/O, persistence, networking, process boundaries
- security/safety controls
- tests and guardrails
- feature participation
- implementation status: implemented / partial / scaffold / unused / inherited / unknown

For every significant feature we eventually trace:
`UI entry → Core command/event/API → Engine subsystem → external/local dependency → state/persistence → result/error → UI`

## Current phase
**Phase 1 — Kinetic Engine**

Status: **STARTED — not yet complete.**

Initial verified observations:
- `kinetic-engine` is a Rust sidecar binary named `kinetic-engine`.
- It has two runtime modes: default MCP stdin/stdout sidecar mode and a `serve` HTTP team-indexer mode.
- `main.rs` declares modules for execution ledger, MCP, model capabilities, code execution agent, orchestration, file enumeration, safety pillars, intent, tools, AI client, audio, vector engine, workspace indexing, symbol extraction/graph, graph retrieval, name index, progress, query engine, embedding, hashing, GPU detection, and indexer.
- Startup applies reduced process priority, loads dotenv, initializes GPU/embedder cache, then dispatches to MCP or team-serve mode.
- Team serve is configured through CLI/environment variables and delegates to `indexer::serve::run`.
- The engine dependency set indicates async execution, HTTP/AI communication, audio capture, LanceDB/Arrow vector storage, FastEmbed embeddings, Git, Tree-sitter symbol extraction, Petgraph GraphRAG, Axum HTTP serving, image/SVG conversion, and optional CUDA support.

## Important rule for resumption
Do not mark a subsystem or file as understood merely because its name, README, or module declaration was inspected. Completion requires reading the implementation and tracing its relationships. If a file cannot be inspected, record it as `BLOCKED` rather than inferring its contents.

## Progress pointer
Current next work: enumerate the complete `kinetic-engine` tree, then inspect files systematically beginning with crate entry/configuration and proceeding through each subsystem and nested directory.
