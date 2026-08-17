# Kinetic Engine — MCP Contract and Cross-Layer Audit

## Status
**PARTIAL — MCP transport/dispatch has now been inspected in detail; final engine closure still requires remaining source-file inventory reconciliation, full tool implementation verification, and tests/build/config audit.**

## MCP bridge architecture

`mcp.rs` is the principal Rust↔TypeScript process boundary. It uses newline-delimited JSON over stdin/stdout rather than an external HTTP transport.

### Engine → host
`McpBridgeClient` owns:

- outbound `mpsc::Sender<Value>`;
- pending request map keyed by request ID;
- atomic request ID counter;
- optional streaming `turn_id`.

Engine-to-host requests are correlated through JSON-RPC-like `id` values and `oneshot` response channels. Gate requests can use a timeout-aware path. A timeout removes the pending sender, preventing a stale late response from leaking state.

Streaming uses explicit messages:

- `stream_token`
- `stream_end`
- `stream_tool_phase_end`

The RAII `StreamingTurnGuard` attaches a turn ID for the duration of an execution plan and clears it on every exit path.

Large stream text is split into UTF-8-safe 512-byte chunks before transmission.

## Host → engine dispatch

`McpServer::listen()` creates:

```text
stdin reader
    ↓
JSON parser
    ↓
notification / response / request discrimination
    ↓
method dispatch
    ↓
Tokio task
    ↓
engine subsystem
    ↓
JSON-RPC response/notification
    ↓
stdout multiplexer
```

The output multiplexer serializes outgoing messages through a bounded channel and flushes stdout for each message.

### Host notifications
Currently handled without an RPC response:

- `interrupt_orchestration`
- `engine_ping`

`interrupt_orchestration` calls the orchestrator interrupt signal. `engine_ping` synchronously emits `engine_pong` to keep the heartbeat responsive.

### Request methods traced

#### `processInstruction`
Primary agent execution entry point. Host supplies:

- prompt/instruction
- mode
- workspace path
- global storage path
- provider base URL/key/model
- optional system prompt
- audio/images
- forensic logging flag
- optional fallback provider
- code-lane timeout
- blueprint approval policy
- output token cap
- context window

The request locks the shared `Orchestrator`, invokes `execute_plan`, and returns `status/outputText`. A special `STREAM_ERROR_EMITTED` error is converted to a handled result to prevent duplicate UI error rendering.

#### `verifyConnection`
Delegates directly to `Orchestrator.ai_client.verify_connection`.

#### `enumerate_workspace_files`
Uses `FileEnumerator`, not the vector indexer. Results are batched in groups of 300 and separated by a 50 ms breathing gap. A final `file_enumeration_complete` result reports totals.

#### `index_workspace`
Uses the canonical `WorkspaceIndexer` through the process-wide single-flight job queue. It lazily initializes the shared `VectorEngine`, runs the build, invalidates retrieval caches on success, and returns `indexed`.

#### `update_file_index`
Lazily initializes the shared VectorEngine and delegates to `WorkspaceIndexer::update_file`; successful mutation invalidates retrieval caches.

#### `delete_file_index`
Removes the old file's indexed chunks through `WorkspaceIndexer`, used for deletion/rename cleanup, then invalidates retrieval caches.

#### retrieval/query method
The query path initializes/reuses the VectorEngine and delegates to `indexer::retrieve::execute_retrieve_with_db`, returning matches plus graph/BM25/vector health and indexed-file count. This is the same retrieval layer used elsewhere rather than a second query implementation.

#### `start_audio` / `stop_audio`
Manage the engine's `AudioState`. Missing default input is represented as a clean disabled result rather than an error.

#### `revise_blueprint`
Calls `Orchestrator::revise_blueprint` with the original plan, user comment, stable blueprint ID, provider credentials, workspace, optional fallback, and bridge client. The stable blueprint ID is preserved across revision so approval identity remains bound.

#### `cancel_pending_sentry`
Host can explicitly remove stale pending gate request IDs. This wakes waiting orchestration calls by dropping the oneshot sender and prevents stale request entries after a new chat, interrupt, or reload.

#### `reset_engine_context`
Directly clears the orchestrator's context buffer. The comments establish that the host is additionally responsible for deleting persisted session history; this RPC intentionally bypasses normal agent execution.

#### `rehydrate_context_buffer`
Host injects saved user/assistant messages into engine memory after session loading. This makes restored messages authoritative before the next Code turn.

#### `report_context_telemetry`
Returns engine-side estimates of tool-schema tokens and context-buffer tokens for the requested lane. This prevents the UI from guessing token consumption for data it cannot measure itself.

#### `compact_replace`
Synchronizes UI compaction with engine context and persistent session history. The engine receives removed IDs, summary text, and global storage path and performs the corresponding context replacement.

#### `record_feedback`
Writes user feedback through the Pillar feedback subsystem using workspace, feedback, context, and session ID.

Unknown methods receive a JSON-RPC method-not-found error.

## Background engine tasks established by MCP startup

`listen()` also starts three important background paths:

1. **Output multiplexer** — serializes all outbound messages.
2. **Index progress forwarder** — forwards bounded `ProgressEvent` values as `kinetic_index_progress` notifications.
3. **Idle vector compactor** — periodically drains the dirty-table set after a quiet window and compacts active LanceDB tables.

A one-shot GPU detection notification (`kinetic_gpu_detected`) is emitted at boot when detector information is available.

## Shared runtime state

The MCP server holds a shared optional `VectorEngine` behind an async mutex. Multiple index/query operations therefore converge on the same initialized engine instance during the process lifetime.

The Orchestrator is separately wrapped in an async mutex and cloned into the request-processing tasks. This provides serialized mutable orchestrator state while allowing the stdin loop to remain responsive.

## Cross-layer invariants

### Canonical indexing

```text
MCP index_workspace
MCP update_file_index
MCP delete_file_index
Team server index trigger
        ↓
WorkspaceIndexer
        ↓
VectorEngine
```

There is no separate MCP-specific vector indexing algorithm.

### Canonical retrieval

```text
MCP query
Team query
Orchestrator retrieval
        ↓
indexer::retrieve
        ↓
QueryEngine + GraphRetriever + VectorEngine
```

### Agent execution

```text
TS host
  ↓ processInstruction
MCP
  ↓
Orchestrator
  ↓
Intent / lane
  ↓
AI provider
  ↓
Tool calls
  ↓
Tools + safety + Sentry
  ↓
Execution ledger / state
  ↓
MCP streaming
  ↓
TS host/UI
```

## Important implementation distinctions

- `enumerate_workspace_files` is deliberately **not** vector indexing.
- Sentry is an engine→host permission boundary, while `tools::command_risk` and `pillars::safety_gate` provide deterministic local safety layers.
- Blueprint approval is a separate policy from Sentry.
- Engine context reset/rehydration is intentionally separated from ordinary `processInstruction` execution.
- Context telemetry is measured by Rust because the engine owns tool schema and context-buffer knowledge.
- Stream errors have explicit duplicate-render prevention semantics.

## Remaining engine work

This file does not mark the engine complete. Remaining closure tasks:

1. Reconcile every file in `src/` against the architecture ledger.
2. Complete every `tools/` implementation and confirm call sites from Orchestrator/MCP.
3. Inspect all `intent/` tests and classifier branches, not only the primary classifier source.
4. Inspect all `pillars/` tests and call sites.
5. Finish all remaining Orchestrator branches and compare its public methods against every caller.
6. Inspect `main.rs`, Cargo/build metadata, tests, scripts, Docker/runtime support, and feature flags for operational architecture.
7. Verify no nested source directory/file has been missed.
8. Perform an end-to-end engine audit and update `00_MASTER_ARCHITECTURE.md` only after these checks.

## Confidence rule

The engine should only be marked **COMPLETE** after the source tree inventory and call-site audit agree with the documented subsystem inventory. A module that exists but is a stub/placeholder must be documented as such rather than counted as implemented functionality.
