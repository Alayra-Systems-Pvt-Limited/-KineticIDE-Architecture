# Kinetic Engine — Rust/TypeScript Stream Boundary and Closure Audit

## Status
**NEAR-COMPLETE.** The current source and stream contract establish Rust as the canonical engine-side structured execution authority, with TypeScript remaining the VS Code bridge/UI authority and compatibility rollback path.

## 1. Stream transport contract

`KINETIC_STREAM_CONTRACT.md` confirms:

```text
Rust engine → child-process stdout → NDJSON
TypeScript bridge → child-process stdin → JSON-RPC 2.0 lines
```

Rust stdout is therefore not one generic JSON-RPC stream. It carries event objects and JSON-RPC notifications/results according to the engine protocol.

Important payloads include:

- `CLASSIFIER_RESULT`
- `MISSION_LIST`
- `blueprint_pulse_review`
- `stream_token`
- `stream_end`
- `stream_error`
- `tool_result`
- `rate_limited`
- `progress` notification
- `sentry/validate` notification

## 2. Turn correlation

`execute_plan` mints a UUID-v4 `turn_id`.

The bridge tracks three correlation dimensions:

```text
turn_id       → logical engine turn
jsonrpc_id    → request/RPC correlation
lane          → effective execution capability
```

The bridge locks `currentLane` from `CLASSIFIER_RESULT` before destructive tools can execute. Calls arriving before lane establishment wait on `laneWaitQueue`; once the classifier result arrives, the queue drains.

This prevents a timing race where a destructive tool payload reaches the bridge before the classifier event.

## 3. Rust canonical tool transition

Current bridge code recognizes a canonical Rust-owned tool set:

- `write_file`
- `read_file`
- `delete_file`
- `execute_command`
- `emit_pulse`
- `propose_edits`
- `web_fetch`
- `get_editor_state`
- `get_diagnostics`

Aliases are normalized before routing:

```text
run_command  → execute_command
create_file  → write_file
```

Canonical parameters are normalized from both `params` envelopes and legacy flat fields.

This is the concrete bridge-side compatibility layer between Rust structured calls and historical payload shapes.

## 4. Lane gate in TypeScript

`emitLaneBlockIfNeeded()` is a second capability boundary after Rust classification.

If the lane is not known, the tool work waits rather than executing immediately.

If a tool is not permitted in the effective lane, the bridge sends a structured JSON-RPC error-like result:

```text
success=false
blocked=true
reasonCode=LANE_CAP
```

The bridge explicitly describes this as an enterprise lane cap, while the Rust side independently computes the lane and tool exposure.

Therefore lane enforcement is deliberately duplicated across the Rust and TypeScript boundary for defense in depth and race prevention.

## 5. Blueprint / approval lifecycle

The bridge maintains approval state per turn rather than a single global boolean:

- approved blueprint turns;
- approved propose-edit paths;
- approved commands;
- pending blueprint JSON-RPC waiters.

`blueprint_pulse_review` can defer the Rust JSON-RPC response until the interactive approval resolves. The bridge can therefore hold the engine request while the VS Code/UI approval lifecycle runs.

Approval state is narrowed to the approved turn and approved paths/commands rather than granting unrestricted mutation access.

## 6. Sentry bridge authority

The bridge maintains a map of pending Sentry requests and a watchdog timer per request. This replaced a single scalar pending request ID so concurrent approval gates cannot overwrite one another.

The architecture explicitly treats the SentryBridge as the single writer for JIT auto-approval paths rather than allowing multiple components to write directly to engine stdin.

This is an important concurrency invariant.

## 7. Stream interruption and stale-turn defense

The bridge uses `StreamGuard`, suppressed turn IDs, and current turn correlation to discard late payloads after:

- user Stop;
- preemption;
- custom-AI idle timeout;
- stream error.

The stream contract documents two observable rejection reasons:

- `stream_ignored_post_user_abort`
- `stream_ignored_stale_turn`

Rust also checks `interrupt_requested` before forwarding stream tokens.

This creates protection on both sides of the process boundary against stale stream data contaminating a new turn.

## 8. Rust-owned sentinel tools

The current default setting is:

```text
kinetic.engine.legacyTsSentinelToolDispatch = false
```

When false, Rust executes the structured sentinel tool batch and emits `stream_end` with `skip_ts_sentinel_tools=true` so the TypeScript bridge does not execute the same batch again.

When true, the bridge injects `KINETIC_LEGACY_TS_SENTINEL_TOOLS=1` into the engine process and restores the older TypeScript sentinel execution behavior.

This is an explicit rollback switch, not the current preferred authority.

## 9. Stream lifecycle

The current contract is:

```text
CLASSIFIER_RESULT
       ↓
MISSION_LIST / optional blueprint
       ↓
stream_token* / tool phases
       ↓
tool_result(s)
       ↓
stream_end OR stream_error
```

`stream_end` controls bridge/UI completion and can suppress TypeScript sentinel dispatch when Rust already executed the tool batch.

The bridge additionally records `TURN_RECORD` telemetry for lane resolution, stream start/end, errors, dispatch, completion, and failure.

Telemetry is disabled by default and can be enabled through `kinetic.telemetry.turnRecords`.

## 10. Index RPC boundary

The bridge has a per-workspace `_indexWorkspacePromises` map to coalesce duplicate `indexWorkspace` requests. It also has a unified per-path debounce queue for update/delete index mutations, preventing independently spawned RPC tasks from reversing user-intent order.

This is an important cross-boundary consistency control because Rust's async execution does not inherently preserve ordering between separate spawned operations.

## 11. Current architectural authority

The verified authority split is now:

```text
                    Kinetic IDE
                         │
             ┌───────────┴───────────┐
             │                       │
       Rust Engine              TypeScript Bridge
             │                       │
     AI/orchestration          VS Code lifecycle
     indexing/retrieval        UI/dashboard
     structured tools          Sentry/UI approval
     intent/lane               stream rendering
     execution ledger           compatibility routing
             │                       │
             └──────── JSON-RPC / NDJSON ────────┘
```

Rust is the preferred authority for structured tool execution. TypeScript remains responsible for VS Code integration and UI-facing effects, with explicit legacy compatibility.

## 12. Engine closure assessment

The major architectural layers have now been directly inspected and connected:

1. Runtime entrypoint and process lifecycle.
2. MCP transport and RPC correlation.
3. Orchestrator and execution loop.
4. Intent classifier and lane ceiling.
5. Tool catalog and policy.
6. Tool executors and safety gate.
7. Indexing and retrieval.
8. Graph/symbol infrastructure.
9. AI/provider client.
10. Execution ledger and progress.
11. Memory/feedback/session state.
12. Team indexer server.
13. Regression guardrails.
14. Rust↔TypeScript stream boundary.

### Still required before the final COMPLETE marker

- Exhaustive 20-tool catalog-to-executor matrix against current `toolCatalog.json` and `toolRouter.ts`.
- Final timeout enforcement matrix.
- Remaining orchestrator branches not already accounted for.
- Current CI/test execution metadata and feature-build verification.
- Final scan of engine source inventory for any unstudied file or nested module.

The engine should remain **NEAR-COMPLETE** until these mechanical closure checks are finished. No architectural assumptions should be promoted to facts beyond the source verified above.
