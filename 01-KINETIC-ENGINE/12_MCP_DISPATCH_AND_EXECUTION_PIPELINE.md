# Kinetic Engine — MCP Dispatch and Execution Pipeline

## Status
**PARTIAL — MCP transport and major execution branches are traced. Final closure still requires the remaining `execute_plan` tail, all tool-call dispatch branches, engine tests/build configuration, and TypeScript bridge cross-check.**

## 1. Engine process boundary

`McpServer::listen()` is the Rust engine's process boundary. It owns newline-delimited JSON on stdin/stdout and multiplexes several classes of traffic:

```text
stdin NDJSON
   ↓
JSON parse
   ├── host response → pending request resolver
   ├── JSON-RPC notification → interrupt / ping
   └── host method request → spawned handler
                              ↓
                         Orchestrator / services
                              ↓
stdout NDJSON multiplexer
```

The stdout writer is centralized in an output task consuming a bounded `mpsc` channel (capacity 100). Every outbound message is serialized and newline-terminated before flush.

## 2. Request/response correlation

`McpBridgeClient` maintains:

- atomic numeric request IDs;
- `pending: HashMap<String, oneshot::Sender<Value>>`;
- outbound `mpsc::Sender<Value>`.

Engine→host RPC flow:

```text
send_custom_request
    ↓
allocate ID
    ↓
insert oneshot sender into pending
    ↓
send JSON payload
    ↓
await response
    ↓
stdin receives matching id
    ↓
remove pending sender
    ↓
resolve caller
```

The timeout-aware variant removes the pending sender on timeout or host disconnect, explicitly preventing stale sender leakage and making late responses harmless.

## 3. Sentry permission RPC

Mutation/command tools use `request_permission()` / the underlying custom request path to send:

```json
{
  "jsonrpc": "2.0",
  "id": <id>,
  "method": "sentry/validate",
  "params": {
    "action": "...",
    "params": { ... }
  }
}
```

A JSON-RPC error becomes a Rust error with the Sentry message. Successful permission returns the `result` object.

This is a **host authorization layer**, while the deterministic Rust safety gate remains an independent precondition for filesystem/command mutation.

## 4. Streaming turn correlation

`StreamingTurnGuard` temporarily stores a `turn_id` in `McpBridgeClient`.

Every streaming token/end event can therefore carry the same turn correlation:

```text
execute_plan
    ↓
StreamingTurnGuard(turn_id)
    ↓
stream_token / stream_tool_phase_end / stream_end
    ↓
TS bridge correlates to same chat turn
    ↓
RAII Drop clears turn_id
```

This is intentionally scoped so a later unrelated request cannot inherit the previous turn's identifier.

`send_stream_end_append_chat()` also carries two important ownership flags:

- `append_chat`
- `skip_ts_sentinel_tools`

These explicitly coordinate whether TS appends the final assistant bubble and whether TS must avoid replaying Rust-owned native tool calls.

## 5. Progress/event side channels

The server starts a bounded index-progress channel and forwards progress as JSON-RPC notifications using method `kinetic_index_progress`.

A separate vector idle-compactor task periodically drains the dirty table set after a quiet window and compacts tables through the active shared `VectorEngine`.

Boot GPU detection emits a one-shot `kinetic_gpu_detected` notification. This is telemetry/coordination; the comment explicitly states the TS bridge owns post-install download UX.

## 6. Host notification handling

Notifications with no `id` are handled without a JSON-RPC response.

Currently observed special notifications:

- `interrupt_orchestration` → signals the orchestrator interrupt flag;
- `engine_ping` → synchronously emits `engine_pong` with a Unix-millisecond timestamp.

The ping path is deliberately synchronous so the stdin loop remains responsive.

## 7. `processInstruction` is the primary host→engine execution entrypoint

The host sends a `processInstruction` method request. Rust extracts:

- prompt/instruction;
- mode;
- workspace path;
- global storage path;
- provider base URL;
- API key;
- model ID;
- custom system prompt;
- audio/images;
- forensic logging flag;
- fallback provider credentials;
- code-lane timeout;
- Blueprint approval policy;
- output token cap;
- context window.

The handler locks the cloned orchestrator and calls `execute_plan(...)`.

Success returns:

```text
jsonrpc result
status = success
outputText = orchestrator result
```

A special `STREAM_ERROR_EMITTED` error is converted into a successful JSON-RPC result with `stream_error_handled` so the UI does not render the same streaming error twice.

All other errors become JSON-RPC errors.

## 8. Other direct host methods

Confirmed direct handlers include:

### `verifyConnection`
Delegates to `AiClient::verify_connection()` and returns the provider/model connectivity result.

### `enumerate_workspace_files`
Uses `FileEnumerator`, not the vector indexer. Results are chunked into batches of 300 with a 50ms breathing gap and emitted through `file_enumeration_batch`; the final JSON-RPC response reports totals.

This is explicitly a non-vector file enumeration service.

### `index_workspace`
Uses the canonical indexer path:

```text
job_queue::run_coalesced
       ↓
shared VectorEngine initialization
       ↓
WorkspaceIndexer::build_index_with_options
       ↓
invalidate retrieval caches
       ↓
status=indexed
```

Concurrent builds for the same workspace/storage key coalesce.

### `update_file_index` / deletion paths
These initialize/reuse the shared `VectorEngine` and delegate file-level mutations to the canonical indexer machinery. The remaining exact branches should be captured from the remainder of `mcp.rs` during final audit.

## 9. Shared VectorEngine lifecycle

`McpServer` keeps:

```text
Arc<TokioMutex<Option<VectorEngine>>>
```

The first index operation creates the engine; subsequent requests clone/reuse the same engine handle.

The idle compactor receives the same shared slot and therefore follows the currently initialized workspace engine.

This is important: vector storage is not reconstructed per query.

## 10. Orchestrator streaming architecture

`send_prompt()` constructs the complete model request:

```text
session context
+ neighbor context
+ memory context
+ feedback context
+ user prompt
        ↓
system prompt by mode
        ↓
health-based dynamic timeout
        ↓
tool schema slice from lane
        ↓
UnifiedRequest
        ↓
AiClient
```

The lane determines the tool schema slice using `build_tool_schemas_for_lane()`.

Health probing dynamically adjusts timeout:

- base lane timeout for fast health;
- +60 seconds when provider RTT is ≥2s;
- failed health probe uses base timeout and a 90-second probe backoff;
- successful timeout decisions are cached for 30 seconds.

Rate-limit handling can emit a `rate_limited` event and retry against the configured fallback provider.

## 11. Rust-owned streaming tool loop

For compatible non-Anthropic endpoints, when legacy TS sentinel handling is disabled and a tool lane is present, the engine can own structured tool execution.

The accumulated streaming response is classified into:

```text
NoStructuredTools
Complete tool payload
Incomplete payload with salvageable calls
Incomplete payload with no salvageable calls
```

This prevents an incomplete `__STRUCTURED_TOOL_CALLS__` marker from being rendered as ordinary chat text.

When structured calls are successfully handled, Rust executes the tools and performs follow-up model rounds. It then sends:

```text
stream_end
append_chat=false
skip_ts_sentinel_tools=true
```

This establishes Rust as the execution authority for that turn and prevents duplicate TypeScript execution.

## 12. Truncated structured-call behavior

If a sentinel is present but truncated and no calls can be safely salvaged:

1. Rust emits a structured `stream_error`.
2. The sentinel is stripped from visible text.
3. User receives a clean interruption message.
4. No partial action is executed.
5. Stream ends with TS sentinel execution disabled.

This is an explicit fail-closed behavior.

## 13. Planning/execution two-stage model

The code-mode/plan execution path has a distinct planning stage and execution stage.

The planning stage can resolve discovery calls such as:

- `get_tool_registry`
- `list_dir`
- `grep_search`
- `crawl_website`
- `take_screenshot`

A discovery result can be injected back into the user's request and planning rerun so the final plan contains real execution tools rather than treating discovery output as the finished task.

After approval, the execution prompt requires a raw JSON array of execution steps.

The execution stream is parsed into tool calls and includes markdown rescue for explicitly named code blocks. Crucially, unnamed fenced blocks are skipped rather than assigned fabricated filenames.

## 14. Blueprint approval boundary

Blueprint approval is distinct from Sentry permission.

```text
AI plan
  ↓
plan_requires_blueprint_gate
  ↓
structural touch/destructive analysis
  ↓
host-selected policy:
   always → mutating plans always gate
   threshold → small/safe plans may auto-continue
  ↓
blueprint_pulse_review RPC
  ↓
Approve / Reject / Revise
```

The revision path reuses the stable original Blueprint ID so approval state remains correlated across revisions.

## 15. Agent execution / auto-heal

The code execution path can detect a failed command and construct an auto-heal prompt.

The recovery loop may produce:

- a replacement `execute_command`;
- a `write_file` corrective action;

and then retry the failed command.

The implementation records execution output and eventually emits a natural-language follow-up summary with tools disabled.

This means the agent can transition:

```text
execute
 ↓
failure
 ↓
auto-heal reasoning
 ↓
corrective tool
 ↓
retry
 ↓
success/final failure
 ↓
natural-language summary
```

The exact retry count and all terminal conditions remain part of the final `execute_plan` tail audit.

## 16. Critical current architecture observation

The repository contains evidence of an ongoing migration from **dual orchestration** toward a clearer Rust-owned execution authority:

```text
Historical / compatibility:
TS stream_end → TS canonical tool dispatch

Current Rust-owned path:
Rust streaming model response
        ↓
Rust parses structured calls
        ↓
Rust executes tools
        ↓
Rust follow-up completion
        ↓
TS receives final stream with skip_ts_sentinel_tools=true
```

Anthropic-native endpoints and legacy sentinel configuration remain explicit compatibility exceptions.

This is not merely a theoretical architecture: the source contains the flags, turn correlation, and stream phase-end protocol needed to enforce the ownership boundary.

## 17. Remaining final-audit items

Before marking Engine complete:

- finish reading the remainder of `mcp.rs` method dispatch;
- finish the complete `execute_plan` tail, including every tool executor branch;
- map all 20 catalog tools to execution paths;
- inspect stream helper/parser functions and tests;
- inspect Cargo manifest/build features and engine tests;
- cross-check `mcpBridge.ts`, `toolRouter.ts`, and `intentWall.ts` against Rust's ownership assumptions;
- verify cancellation behavior across every asynchronous execution path;
- verify timeout behavior at every layer, not just `McpBridgeClient`;
- document exact environment variables and provider contracts.

Until those checks are completed, the engine remains **PARTIAL**.
