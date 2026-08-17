# Kinetic Engine — Entry Point and Runtime Boundary

Source revision: `bd8a1b378d7142e0e6e97d6619c3b4ac315bc29c`

## 1. Crate identity
`extensions/kinetic-core/kinetic-engine/Cargo.toml` defines a Rust 2021 crate named `kinetic-engine`, version `0.1.0`, described as the high-performance sidecar engine for Kinetic IDE.

## 2. Runtime modes
The binary has two explicit subcommands:

- `mcp`: MCP stdin/stdout sidecar used by the IDE.
- `serve`: shared/team indexer HTTP server under `/api/v1/indexer/*`.

When no subcommand is supplied, `main()` defaults to `Mcp`.

## 3. Startup sequence
Verified sequence in `main.rs`:

```text
process priority adjustment
        ↓
dotenv loading
        ↓
CLI parse
        ↓
GPU + embedder cache initialization
        ↓
command dispatch
   ┌────┴────┐
   ↓         ↓
  MCP      TEAM SERVE
```

### Process priority
- Windows: `BELOW_NORMAL_PRIORITY_CLASS` through `windows-sys`.
- Linux/macOS: `libc::nice(10)`.

This intentionally makes the sidecar lower priority than foreground IDE work.

### GPU/embedder initialization
`init_gpu_and_embedder_cache()`:
- reads `KINETIC_GLOBAL_STORAGE`;
- warns when it is unset and FastEmbed cache therefore falls back to `.fastembed_cache/` in the current working directory;
- calls `gpu_detector::detect()`;
- when CUDA is active and the expected DLL exists, sets `ORT_DYLIB_PATH`;
- stores boot GPU information using `gpu_detector::set_boot_info()`;
- logs the canonical vector-index pipeline: `WorkspaceIndexer → Lance` and separately identifies `FileEnumerator` as Vault-only enumeration.

This is an important architectural boundary: the entry point itself does not construct the vector index. It initializes environment/runtime prerequisites and delegates to subsystems.

## 4. MCP boundary
`run_mcp_sidecar()` creates:
- `mcp::McpServer`
- `orchestrator::Orchestrator`

Then calls `McpServer::listen(&mut orchestrator)`.

This establishes the primary execution chain as:

```text
TypeScript host / IDE
        │ JSON lines over stdin/stdout
        ▼
     McpServer
        │
        ▼
   Orchestrator
        │
        ├── intent
        ├── safety/pillars
        ├── code execution
        ├── tools
        ├── AI client
        └── indexing / retrieval / state subsystems
```

The exact downstream call graph remains to be traced from implementation files.

## 5. MCP server observations already verified
`mcp.rs` is substantially more than a transport wrapper.

### McpBridgeClient
It owns:
- outbound `mpsc::Sender<Value>`;
- pending JSON-RPC request map keyed by generated IDs;
- atomic request ID generator;
- optional streaming `turn_id` shared state.

It supports:
- permission validation through `sentry/validate`;
- progress notifications;
- arbitrary custom payloads;
- stream phase-end signalling;
- UTF-8-safe 512-byte stream chunking;
- stream-end metadata controlling chat append behavior and whether TS sentinel tool calls should be skipped;
- timeout-aware host RPC (`send_custom_request_with_timeout`).

### Host-RPC timeout hardening
Non-zero RPC timeouts use `tokio::time::timeout`. On timeout or host disconnect, the corresponding pending oneshot sender is removed. The source comments explicitly identify this as a fix for an engine task potentially hanging forever when the bridge dies during a gate/RPC.

### Streaming correlation
`StreamingTurnGuard` attaches a `turn_id` to stream tokens/end events for the lifetime of an execution-plan scope and clears it through `Drop`.

### McpServer owned runtime state
The server contains:
- `FileEnumerator` for non-vector file enumeration;
- shared `AudioState`;
- shared optional `VectorEngine`.

### Background tasks started by `listen()`
The observed implementation starts at least:
1. stdout output multiplexer;
2. bounded progress-event forwarding channel/task;
3. idle vector-table compactor task;
4. GPU detection notification forwarding task/state handling.

The progress channel is bounded and uses the engine's configured capacity, explicitly replacing an older unbounded design to control memory pressure.

The compactor drains dirty tables after a quiet window, snapshots the currently initialized `VectorEngine`, opens marked Lance tables, and compacts them.

### Host input loop
`listen()` reads stdin line-by-line, ignores empty lines, parses JSON values, and handles JSON-RPC notifications such as `interrupt_orchestration`. The complete dispatch table and request handlers still need to be traced in the remainder of `mcp.rs`.

## 6. Indexer server boundary
`run_team_serve()` translates CLI/environment configuration into `indexer::config::ServeConfig`, then delegates to `indexer::serve::run()`.

Known configuration inputs:
- `KINETIC_SERVE_BIND` (default `0.0.0.0:9100`)
- `KINETIC_TEAM_WORKSPACE`
- `KINETIC_TEAM_DATA_DIR`
- `KINETIC_TEAM_REPO_ID`
- `KINETIC_INDEXER_API_KEY`
- optional `KINETIC_TEAM_COMPANY_ID`
- `KINETIC_TEAM_GIT_POLL_SECS` (default 300)

The full HTTP/auth/job/retrieval flow will be documented in the Indexer subsystem study.

## 7. Dependency architecture observed from manifest
The crate combines several major subsystems:

- Async/runtime: Tokio, Futures.
- AI/network: Reqwest, Serde JSON, dotenvy.
- Local semantic index: LanceDB, Arrow, FastEmbed.
- Code intelligence: Tree-sitter for Rust/TypeScript/JavaScript/Python/Go/Java, Petgraph.
- Workspace/Git: ignore, globset, Git2, SHA-256, UUID, path utilities.
- Team HTTP service: Axum, Tower, Tower HTTP.
- Audio: CPAL, Hound, Base64.
- Vision: image, resvg, usvg, tiny-skia, URL encoding.
- Hardware: sysinfo + optional FastEmbed CUDA.

These dependencies describe capabilities but are not treated as proof of runtime wiring until their implementation and callers are inspected.

## 8. Initial architecture conclusion
The engine is a **multi-role local sidecar**, not merely an AI HTTP client. Its runtime boundary combines:

1. IDE transport/bridge (MCP JSONL);
2. orchestration and tool execution;
3. safety/permission controls;
4. AI provider communication;
5. local code/workspace indexing;
6. vector + graph retrieval;
7. audio/vision capabilities;
8. GPU/runtime capability detection;
9. optional team-shared indexer HTTP service.

This conclusion is provisional until the complete file-by-file study is finished.

## Status
**PARTIAL — entry point and manifest studied; full engine not yet studied.**

Next: finish MCP dispatch, then orchestrator/code execution, safety pillars, intent, tools, indexing, retrieval, AI client, and all remaining root/nested files.
