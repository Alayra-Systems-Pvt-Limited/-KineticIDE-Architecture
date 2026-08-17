# Kinetic Engine — Orchestration, MCP Bridge, Intent and Tool Control

## Study status
**PARTIAL.** The central control surfaces have been inspected, but `orchestrator.rs` is very large and still requires complete function-by-function tracing and cross-reference validation before the engine can be declared complete.

## 1. Orchestrator role

`orchestrator.rs` is the central Rust execution coordinator. It owns or coordinates:

- AI client access (`UniversalClient`);
- rolling context (`VecDeque<Value>`);
- session context (`SessionContext`);
- session trust/distillation state;
- provider health timeout cache;
- cooperative interruption;
- host-supplied code-lane timeout;
- blueprint approval policy;
- output-token/context-window budgets;
- Sentry and blueprint gate timeouts.

This makes it substantially more than a planner: it is the execution state machine connecting AI responses, structured tool calls, host gates, tool execution, streaming, and session state.

## 2. Streaming structured-tool protocol

The orchestrator implements a Rust-owned parser around the structured tool-call sentinel:

`__STRUCTURED_TOOL_CALLS__`

It explicitly handles three states:

```text
None        → no sentinel → normal prose
Complete    → complete structured calls
Incomplete  → sentinel present but payload truncated
```

The parser has a salvage path that scans top-level JSON objects while respecting strings, escapes, and nesting. Valid leading tool calls can therefore survive a truncated stream.

This is paired with a scrubber so the sentinel or JSON tail cannot accidentally become visible chat prose.

The design explicitly prevents a truncated structured stream from falling through to the ordinary plain-text rendering path.

## 3. Tool-call follow-up control

A streaming turn has a bounded follow-up limit:

`STREAMING_TOOL_FOLLOW_UP_MAX_ROUNDS = 8`

The intended control flow is:

```text
AI stream
  ↓
structured sentinel?
  ├─ no → prose stream
  └─ yes
       ↓
parse / salvage
       ↓
execute Rust-owned tool calls
       ↓
stream_tool_phase_end
       ↓
AI follow-up
       ↓
repeat, max 8 rounds
```

This is a deterministic bound against an endless tool/assistant loop.

## 4. Factual execution narrative

`execution_ledger.rs` is integrated conceptually with orchestration. It creates user-visible narratives from completed tool receipts rather than model claims.

The ledger distinguishes successful and failed steps and can replace a model summary when the model falsely claims that no writes occurred despite successful execution receipts.

This is an important trust boundary:

```text
actual tool receipt
      ↓
execution ledger
      ↓
summary consistency check
      ↓
user-visible result
```

## 5. Mission / Trace pulse system

The orchestrator builds shared Mission and Trace XML pulse bodies used by both the agent loop and legacy planner.

Mission steps include mutating/terminal operations such as:

- `write_file`
- `execute_command`
- `delete_file`
- `replace_in_file`
- `move_file`
- `create_directory`
- `propose_edits`

Trace events contain actual execution state and optionally a real command exit code. The implementation intentionally avoids fabricating forensic fields.

Trace rollup is:

```text
failed + done → partial
failed only   → error
done only     → ok
```

XML values are escaped before insertion.

## 6. Blueprint approval boundary

`ensure_blueprint_rpc_approved()` enforces host approval when approval is required. A blueprint is considered approved only when `approved == true` or `ok == true`.

Otherwise orchestration fails before mutating tools execute.

The orchestrator has an explicit approval-policy setting, defaulting to strict `always`, with a `threshold` alternative supplied by the host.

The implementation explicitly keeps blueprint approval independent from the Sentry Gate state.

## 7. Health-aware request timeouts

The orchestrator contains a TTL cache of provider base URL health-probe results and derives request timeouts from measured RTT. Failed probes receive a backoff window.

This is an execution reliability mechanism, not merely telemetry: the resolved timeout becomes part of outbound AI request behavior.

## 8. MCP bridge architecture

`mcp.rs` provides the Rust ↔ TypeScript process bridge.

`McpBridgeClient` contains:

- Tokio `mpsc::Sender<Value>` for outbound messages;
- pending JSON-RPC request map keyed by IDs;
- atomic request ID generation;
- current streaming turn ID.

Requests are correlated through `oneshot` channels.

### Request flow

```text
Rust orchestrator
      ↓
McpBridgeClient
      ↓
JSON message over stdout
      ↓
TypeScript bridge / host
      ↓
response
      ↓
pending[id]
      ↓
oneshot
      ↓
Rust caller
```

## 9. MCP request timeout and cancellation

`send_custom_request_with_timeout()` supports:

- `timeout_ms == 0` → legacy/unbounded wait;
- nonzero timeout → Tokio timeout;
- pending-map cleanup on timeout;
- late host responses become harmless because the sender is removed.

This was added specifically to prevent engine tasks from hanging indefinitely when a host gate stops responding.

The host can also send `cancel_pending_sentry` to remove stale pending gate IDs after a new chat, interrupt, or extension reload.

## 10. Streaming transport contract

The bridge sends:

- `stream_token`
- `stream_tool_phase_end`
- `stream_end`

Streaming messages may carry `turn_id` for TypeScript correlation.

`StreamingTurnGuard` provides RAII cleanup so the turn ID is cleared when an execution scope exits.

`emit_stream_tokens_chunked()` emits UTF-8-safe 512-byte chunks.

`stream_end` can carry:

- `append_chat`;
- `skip_ts_sentinel_tools`;
- `turn_id`.

This supports Rust-owned native tool execution while preventing the TS bridge from executing the same sentinel tool calls a second time.

## 11. MCP server lifecycle

`McpServer::listen()` establishes:

1. outbound message channel;
2. stdout multiplexer;
3. bounded index-progress channel;
4. global progress initialization;
5. background idle vector-table compactor;
6. progress forwarding task;
7. optional GPU boot notification;
8. stdin JSON-line reader;
9. cloned shared `Orchestrator` state;
10. shared VectorEngine slot.

The server is therefore the process-level integration boundary between the Rust engine and the TypeScript extension host.

## 12. Progress backpressure

Indexer progress is forwarded through a bounded channel rather than an unbounded queue. This protects the engine from unbounded memory growth when the host consumer falls behind.

## 13. Vector compaction integration

The MCP server starts an idle compactor task. It periodically drains dirty vector tables after a quiet window, snapshots the active VectorEngine, and compacts marked tables.

This is deliberately asynchronous and best-effort: unavailable/uninitialized engines and tables are skipped rather than blocking the main bridge.

## 14. Host-driven engine lifecycle methods

The MCP dispatch includes explicit host control for:

- `interrupt_orchestration`;
- `cancel_pending_sentry`;
- `reset_engine_context`;
- `rehydrate_context_buffer`;
- `report_context_telemetry`;
- `compact_replace`;
- `record_feedback`.

These show that engine memory is synchronized with host session state rather than being an isolated Rust-only conversation store.

`reset_engine_context` clears the engine buffer; the host is responsible for deleting its persisted session history as well.

`rehydrate_context_buffer` injects saved messages so the next execution turn sees the rehydrated context as authoritative.

`compact_replace` updates engine memory and the persisted session history as a dual-store operation.

`record_feedback` feeds the workspace/session feedback signal system.

## 15. Intent system

`intent/classifier.rs` defines the single routing vocabulary:

### TaskIntent

- `Informational`
- `Explore`
- `Artifact`
- `CodeChange`
- `VisionClone`

### ExecutionLane

- `AskResponse`
- `PlanArtifact`
- `CodeExec`
- `VisionClone`

The capability model is explicitly:

```text
UserMode × TaskIntent → ExecutionLane
```

The user mode is a capability ceiling; classification cannot exceed it.

Examples:

- `ask` always clamps to `AskResponse`;
- `plan` allows artifacts but caps code changes to planning;
- `code` allows code execution but caps vision clone to code execution;
- `vision` permits the full vision lane.

Unknown/default mode is treated as code mode for the classifier's lane resolution.

## 16. Intent classification signals

Classification combines three signals:

1. structural language;
2. workspace context;
3. session momentum.

Structural signals detect questions, vision markers, code imperatives, planning language, exploration verbs, and casual conversation.

Workspace signals use diagnostics and active-file extension as code-context evidence.

Momentum uses recent intent history to avoid unnecessary lane switching during a continuing coding session.

Attachment logic adds a safety override: analytical/review language with an attachment stays informational; a generic attachment can become `VisionClone`.

The classifier also contains targeted safeguards for plan-mode Markdown artifacts and incidental use of words such as “plan” inside implementation requests.

## 17. Mission builder

`intent/mission_builder.rs` converts prompts into a simple mission list for UI/execution presentation.

It detects:

- package installation;
- mentioned files;
- create vs modify semantics;
- general fallback missions.

Mission types are:

- `CreateFile`
- `ModifyFile`
- `RunCommand`
- `InstallPackage`
- `General`

The mission builder deliberately does **not** perform the main intent classification; that responsibility is centralized in `classifier.rs`.

## 18. Tool execution boundary

`tools/mod.rs` is the low-level workspace operation surface. It exposes read/write, command, search, website crawling/screenshot support, project serving, debug-state persistence, and related helpers.

Important safety sequence for mutations:

```text
AI/tool request
      ↓
local deterministic safety gate
      ↓
host Sentry permission
      ↓
filesystem/process mutation
      ↓
result / ledger / progress
```

`write_file` invokes the local safety gate before requesting host permission and writing.

`execute_command` likewise checks the local command safety gate before asking Sentry permission and spawning the platform shell.

The command shell is:

- Windows: `cmd /C`;
- non-Windows: `sh -c`.

## 19. Tool surface versus policy

The tools module exports separate policy/schema functions, including:

- lane-specific tool availability;
- mode policy parsing;
- workspace-relative path normalization;
- forbidden-write detection;
- workspace-relative write validation;
- Sentry tier lookup;
- tool timeout lookup.

This separates **what a tool can do** from **when a lane is allowed to offer it**.

## 20. Important implementation boundaries discovered

Confirmed stubs remain in the engine:

- `bridge.rs` — placeholder; real bridge is `mcp.rs`;
- `infrastructure_master.rs` — future infrastructure supervisor placeholder.

The architecture documentation must not describe those placeholders as functioning subsystems.

## 21. Current engine architecture picture

```text
                 TypeScript Extension Host
                         │
                  JSON / MCP bridge
                         │
                         ▼
                 ┌───────────────┐
                 │  MCP Server   │
                 └───────┬───────┘
                         │
                 ┌───────▼────────┐
                 │  Orchestrator  │
                 └───┬─────────┬───┘
                     │         │
             ┌───────▼───┐ ┌──▼──────────┐
             │ AI Client │ │ Intent/Lane │
             └───────┬───┘ └──────┬──────┘
                     │             │
                     └──────┬──────┘
                            ▼
                     Tool execution
                            │
                  ┌─────────┴─────────┐
                  │                   │
             Safety Gate          Sentry
                  │                   │
                  └─────────┬─────────┘
                            ▼
                    Workspace mutation
                            │
                    Execution Ledger
                            │
                       Mission/Trace

Separate context/index path:

Workspace → Indexer → Vector/Graph/BM25 → Retrieval → Orchestrator context
```

## 22. Remaining work before Engine completion

The engine is not yet considered complete. Remaining deep-study targets include:

- all `Orchestrator` execution functions and call graph;
- complete MCP method dispatch, not only lifecycle/control methods;
- every file under `tools/`, including command risk, policy, schemas, image generation, vision assets, and browser-related files;
- all `pillars/` modules;
- complete feedback/context/session behavior;
- any tests and regression guardrails tied to orchestration;
- exact cross-layer TypeScript contracts referenced by Rust;
- final file inventory and dead/stub/unused classification.
