# Kinetic Engine — MCP Bridge, JSONL Transport, and Dispatch

Source: `KineticIDE/extensions/kinetic-core/kinetic-engine/src/mcp.rs` on `main`.

## Transport
The default engine runtime is a Rust sidecar communicating with the TypeScript host over JSON lines on stdin/stdout. `McpServer::listen()` owns the stdin loop; a Tokio output multiplexer serializes outbound JSON values and flushes stdout.

```text
TS host / extension
      ↕ JSONL / JSON-RPC
McpServer + McpBridgeClient
      ↕
Orchestrator / AI / safety / tools
      ↕
Indexer / VectorEngine / retrieval
```

## McpBridgeClient
`McpBridgeClient` contains an outbound `mpsc::Sender<Value>`, a pending request map of generated IDs to oneshot senders, an atomic request-ID counter, and optional streaming `turn_id` state.

`request_permission()` sends `sentry/validate` and waits for the matching host response. `send_custom_request_with_timeout()` performs engine→host RPC with optional hard timeout and removes pending entries on timeout/disconnect, preventing leaked waiters and making late responses harmless. The legacy `send_custom_request()` remains unbounded by passing `timeout_ms = 0`.

## Streaming
The bridge emits `stream_token`, `stream_tool_phase_end`, and `stream_end`. `StreamingTurnGuard` attaches a `turn_id` during an execution scope and clears it through `Drop`. Text is chunked into UTF-8-safe 512-byte pieces. `stream_end` carries `append_chat` and `skip_ts_sentinel_tools`, allowing Rust-native tool execution to coexist with the older TS sentinel compatibility path.

## Server state and background tasks
`McpServer` owns a `FileEnumerator`, shared `AudioState`, and `Arc<TokioMutex<Option<VectorEngine>>>`. VectorEngine initialization is lazy per workspace/storage request.

`listen()` starts an output multiplexer, bounded progress forwarding, an idle vector-table compactor, and GPU boot notification handling. Progress uses a capacity-256 channel; `progress.rs` throttles ordinary `IndexProgress` events to about one per 250ms per phase and exposes dropped-event telemetry. The compactor polls every 30s and only compacts dirty tables after a 10s quiet window, replacing inline compaction after every write.

## Notifications and response correlation
Notifications without an `id` include `interrupt_orchestration`, which signals the shared orchestrator, and `engine_ping`, which synchronously emits `engine_pong` with a Unix-millisecond timestamp.

Messages with an `id` plus `result`/`error` are treated as responses to pending engine→host requests. The matching oneshot sender is removed before delivery.

## Main `processInstruction` request
The host sends prompt, mode, workspace/global storage paths, provider URL/key/model, optional system prompt, audio/images, forensic logging, optional fallback credentials, blueprint approval policy, code-lane timeout, output token budget, and context-window budget. The handler normalizes the blueprint policy to `always` or `threshold`, locks the shared orchestrator, and calls `execute_plan(...)`.

A successful call returns JSON-RPC `{status: success, outputText}`. If Rust already emitted a stream error, the handler returns a handled result instead of producing a duplicate UI error.

## Other important dispatch boundaries
- `verifyConnection` delegates to `UniversalClient::verify_connection()`.
- `enumerate_workspace_files` delegates to `FileEnumerator`; it emits 300-file batches and a completion response. It is explicitly separate from vector indexing.
- `index_workspace` uses `indexer::job_queue::run_coalesced`, lazily initializes VectorEngine, invokes `WorkspaceIndexer::build_index_with_options`, then invalidates retrieval caches.
- `update_file_index` calls `WorkspaceIndexer::update_file` and invalidates retrieval caches.
- `delete_file_index` calls `WorkspaceIndexer::delete_file` and invalidates retrieval caches.
- `retrieve_context` and `query_codebase` use `indexer::retrieve::execute_retrieve_with_db` and expose vector/BM25/graph/cache/revision metadata.
- `get_index_health` aggregates schema versions, index counts, Git revisions, embedder state/cache, graph hardening counters, oversize/file-cap counters, progress backpressure, dirty compaction state, parse failures, and build timestamps. It is intentionally best-effort.
- `start_audio` operates the shared `AudioState` and returns started/disabled/error.
- `cancel_pending_sentry` removes stale pending RPC IDs so waiting tasks wake through dropped oneshots.
- `reset_engine_context` directly clears orchestrator memory; it does not route through the AI path.
- `rehydrate_context_buffer` injects saved transcript messages into engine memory.
- `report_context_telemetry` reports estimated tool-schema and context-buffer token usage.
- `compact_replace` synchronizes engine context/disk session history after UI transcript compaction.
- `record_feedback` records preference feedback through the feedback pillar.
- Blueprint revision requests pass provider credentials, workspace, original plan/comment, stable blueprint ID, fallback credentials, and the bridge into the orchestrator.

## Canonical indexing boundary
The MCP layer is a protocol/orchestration boundary, not the embedding implementation. The canonical semantic path is:

`index_workspace/update_file_index/delete_file_index → WorkspaceIndexer → VectorEngine/LanceDB`.

The separate enumeration path is:

`enumerate_workspace_files → FileEnumerator → Vec<relative paths>`.

The source comments explicitly maintain this separation because Vault secret scanning and semantic indexing have different ignore policies and consumers.

## Architectural conclusion
`mcp.rs` is the primary process boundary and operational coordinator of the engine. It combines JSONL transport, request correlation, streaming, host permissions, cancellation, context lifecycle, indexing/retrieval dispatch, progress delivery, compaction scheduling, GPU telemetry, health telemetry, and audio control. It is substantially more than a thin MCP wrapper.

## Status
**PARTIAL.** MCP transport and major dispatch paths are documented; the full dispatch table and every downstream subsystem still need to be traced before the engine can be marked complete.
