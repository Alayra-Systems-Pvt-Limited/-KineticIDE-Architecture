# Kinetic Engine — MCP Runtime and Orchestrator Continuation

## Status
**PARTIAL.** This pass advances the transport/runtime study and confirms several cross-cutting invariants. The orchestrator is large and still requires branch-by-branch verification before being marked complete.

## MCP bridge contract

`mcp.rs` defines the Rust↔TypeScript transport boundary through `McpBridgeClient` and `McpServer`.

`McpBridgeClient` maintains an outbound async channel, pending JSON-RPC response senders, an atomic request ID, and an optional streaming turn ID. Engine-originated RPCs therefore use request IDs and oneshot response channels rather than polling.

`request_permission()` sends JSON-RPC `sentry/validate`. `send_custom_request_with_timeout()` provides generalized engine→host RPC with optional hard timeout; on timeout it removes the pending sender and returns `sentry_timeout`, preventing stale pending channels.

## Streaming correlation

`StreamingTurnGuard` uses RAII to attach a `turn_id` to streaming events and clear it on scope exit. Streaming messages include the turn ID where available: `stream_token`, `stream_end`, and `stream_tool_phase_end`.

`emit_stream_tokens_chunked()` splits large text into 512-byte UTF-8-safe chunks. `send_stream_end_append_chat()` explicitly tells TypeScript whether final text should append to chat and whether TS sentinel-tool execution must be skipped. This is part of the Rust-owned tool-loop migration.

## MCP server runtime

`McpServer::listen()` establishes the stdio JSON transport:

```text
stdin → JSON parser → method dispatch → Orchestrator / VectorEngine / file enumeration / audio → outbound channel → stdout
```

The server owns `FileEnumerator` for non-vector consumers, shared `AudioState`, and an async optional shared `VectorEngine` slot. The source comments explicitly distinguish `FileEnumerator` from the canonical vector indexing pipeline: `WorkspaceIndexer` remains the vector indexing surface.

## Progress and maintenance

The server initializes the bounded global progress channel and forwards index events as `kinetic_index_progress` JSON-RPC notifications. This creates a backpressure-aware path from WorkspaceIndexer to the TypeScript bridge.

Startup also launches an idle VectorEngine compactor. It drains dirty tables only after the configured quiet window and compacts each marked LanceDB table. It shares the optional VectorEngine slot, so it can observe a workspace initialized later.

A one-shot `kinetic_gpu_detected` notification may be emitted at boot when GPU detection reports information. The source comments classify this as telemetry/coordination rather than core execution behavior.

## Orchestrator streaming continuation

The later orchestrator path confirms the Rust-owned streaming decision:

```text
provider stream
   ↓
full response
   ↓
rust_owned?
 ├─ yes → resolve_streaming_turn_tools_rust_owned
 │          ├─ RustHandled → visible text + stream_end
 │          ├─ NoStructuredTools → safe fallback
 │          └─ ToolCallsTruncated → structured error + no action
 └─ no → legacy / Anthropic-compatible path
```

A truncated structured-tool sentinel is explicitly prevented from reaching the chat renderer. The engine emits a `stream_error`, strips sentinel markers, and tells the user that no partial actions were applied.

## Empty-workspace bootstrapping

`resolve_empty_workspace()` detects an absent/invalid workspace and creates a default project location: Windows uses `Documents/Kinetic-IDE`; Unix-like systems prefer `Documents/Kinetic-IDE` and fall back to `.kinetic/workspace`. It can derive a project folder from a command hint, creates the directory, emits `open_workspace`, and updates the active workspace.

## Blueprint revision

`revise_blueprint()` preserves the original approval identity when a blueprint is revised. A supplied `bp_{turn_id}` is rebound to the same stable blueprint ID. Revised output includes the plan, approval requirement, touch estimate, narrative, user notes, revision provenance, and timestamp. Truncated revised plans fail closed with no changes.

Blueprint policy is normalized to `always` or `threshold`.

## Execute-plan initialization

`execute_plan()` resets interruption state, captures host code-lane timeout, normalizes blueprint approval policy, stores output/context budgets, sanitizes multimodal inputs against model capabilities, clears pillar state, and loads persistent session history from `<global_storage_path>/session_history.json` when present.

The old file-list indexer pass-through was removed; source comments identify `WorkspaceIndexer` as the canonical indexing surface.

## Execution self-healing

The code execution path includes a bounded three-attempt auto-heal loop. On `[EXECUTION FATAL]` it emits degraded health, pauses for UI synchronization, records debug state, inspects a recently modified file, constructs an interrupt prompt with failure/file context, records an `auto_heal` feedback correction, asks for a surgical `write_file` or `execute_command` repair, and retries. After three attempts it stops.

Successful execution emits healthy state and a launch-reveal payload, then asks for a plain-language summary with tools disabled.

## Discovery planning loop

Legacy planning can execute discovery tools (`get_tool_registry`, `list_dir`, `grep_search`, `crawl_website`, `take_screenshot`) first and feed their results back into planning before mutating tool calls are produced. This is distinct from the Rust-owned native tool loop.

## Remaining engine estimate

This is a conservative estimate by **study effort**, not raw line count:

- Core transport/index/retrieval/storage: ~85%
- Orchestrator: ~60–70%
- MCP dispatch/runtime: ~55–65%
- Tools subsystem: ~20–30%
- Intent subsystem: ~0–10%
- Pillars subsystem: ~15–25%
- Remaining root support/tests/build/config: ~25–35%

Overall engine understanding is approximately **55–65% complete** at this checkpoint.

The engine will not be declared complete until every source file/subdirectory has been enumerated and inspected and major cross-file execution flows have been reconciled.

## Next order

1. Finish remaining `orchestrator.rs` branches and public methods.
2. Finish `mcp.rs` dispatch methods and every MCP method contract.
3. Study `tools/` completely, including nested modules and safety boundaries.
4. Study `intent/` completely.
5. Study `pillars/` completely.
6. Reconcile support modules, tests, Docker/build/configuration, and unused/dead paths.
7. Perform final engine file-inventory and end-to-end flow audits.
