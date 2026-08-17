# Kinetic Engine — Orchestrator Control Plane (Part 1)

## Source
`KineticIDE/extensions/kinetic-core/kinetic-engine/src/orchestrator.rs`

## Status
**PARTIAL — parser/policy/pulse foundations inspected. Full turn lifecycle, streaming loop, tool execution, fallback handling, and public orchestration entry points remain to be traced.**

## 1. Role
`Orchestrator` is a central Rust execution controller connecting the host/MCP boundary, `UniversalClient`, `Tools`, `SessionContext`, intent execution lanes, Blueprint approval, Sentry/approval RPCs, Mission/Trace pulses, execution ledger, and workspace mutations. Its state includes context buffering, AI client, session context/trust, health-timeout cache, cooperative interruption, host code-lane timeout, Blueprint approval policy, output-token/context-window budgets, and gate timeouts.

## 2. Streaming structured-tool protocol
The engine parses the `__STRUCTURED_TOOL_CALLS__` sentinel and optional end marker with three states:

- `None`: no sentinel; genuine prose.
- `Incomplete`: sentinel present but payload truncated; complete leading calls may be salvageable.
- `Complete`: finished/parseable payload, including valid zero-call output.

`salvage_leading_tool_calls()` scans top-level JSON objects with string/escape/nesting awareness and keeps complete objects having a non-empty `function.name`. This prevents one truncated trailing call from discarding already-complete calls.

`strip_sentinel_markers()` defensively removes sentinel/end-marker/JSON tail before chat rendering. A legacy `split_streaming_sentinel_tool_calls()` wrapper remains for compatibility. `KINETIC_LEGACY_TS_SENTINEL_TOOLS` enables the legacy TS sentinel path only when explicitly truthy.

## 3. Execution lanes
Tool lane keys map to `ExecutionLane`:

- `ask_response` → `AskResponse`
- `plan_artifact` → `PlanArtifact`
- `vision_clone` → `VisionClone`
- default → `CodeExec`

Asset tools are explicitly permitted only in `CodeExec`/`VisionClone`, and only `generate_image` / `convert_asset`. Blocked responses carry `blocked`, `LANE_CAP`, lane, and reason. This is an engine-side capability boundary.

## 4. JSON and Blueprint safety
`extract_json_objects()` extracts top-level JSON from arbitrary model text using brace counting and serde validation. It is intentionally less defensive than the structured sentinel parser because it is not string-aware.

`blueprint_plan_payload_truncated()` detects unclosed JSON arrays/objects or an unfinished string. The contract is fail-closed: a truncated plan must never become prose, auto-approve, or mutate.

`resolve_markdown_plan_from_plan_object()` normalizes possible plan bodies (`markdown_plan`, `plan`, `body`, `content`, `markdown`, `steps`, `outline`) and emits a bounded diagnostic if no usable body exists.

Plan tool names normalize from `tool`, `function.name`, or `name`; arguments normalize from OpenAI-style `function.arguments`, `arguments` string/object, or the complete call object.

## 5. Blueprint gate policy
Mutating/external tools identified for Blueprint review include:

`write_file`, `propose_edits`, `execute_command`, `delete_file`, `replace_in_file`, `move_file`, `create_directory`, `generate_image`, `convert_asset`, `capture_screenshot`, `ui_interact`, `web_fetch`.

Policy touch accounting understands multiple `propose_edits` shapes and counts distinct paths, maximum operations on one path, and total operations. Threshold review occurs for ≥3 distinct files, ≥4 operations on one file, or destructive/terminal tools. Reasons include `destructive_or_terminal_step`, `multi_file_threshold`, `single_file_edit_burst`, `no_file_writes`, and `below_threshold_auto`.

The host-threaded policy is applied in one Rust location because Rust does not read VS Code configuration:

- `threshold`: use structural heuristics; small/safe plans can auto-continue.
- `always`: strict mode; every mutating plan requires explicit approval.

The source explicitly keeps Blueprint policy independent of the Sentry Gate.

## 6. Blueprint narrative and approval
`build_blueprint_narrative()` derives `intro` and `verification` directly from the existing plan, without another AI call. It describes file/change counts, deletions, commands, test/build indicators, and verification guidance.

`ensure_blueprint_rpc_approved()` fails closed when approval is required. It accepts `approved=true` or `ok=true`; otherwise execution stops before mutating tools and includes the returned reason.

## 7. Mission / Trace
The orchestrator is a shared Rust-side builder for Mission/Trace pulse bodies used by both the newer `code_exec_agent` path and the legacy planner. TypeScript `PulseProcessor` remains the schema SSOT.

Mission steps are visible mutating/terminal operations: `write_file`, `execute_command`, `delete_file`, `replace_in_file`, `move_file`, `create_directory`, `propose_edits`.

Mission emits an intro plus pending `<step>` records. Trace emits an intro plus executed `<event>` records with phase, tool, file, `done`/`failed` state, narrative, optional command/output, and real exit code when available. `healed` is deliberately not fabricated.

`trace_rollup_status()` produces `ok`, `partial`, or `error`. `xml_escape()` protects XML content. `pulse_intro_trim()` performs bounded sentence-aware trimming while avoiding common abbreviation/file-extension/decimal false stops.

## 8. Current execution architecture inferred from this section
```text
Model stream
   ↓
Sentinel classification
   ├── complete
   ├── incomplete + salvage
   └── none
   ↓
Normalized tool calls
   ↓
Execution-lane capability filter
   ↓
Blueprint structural policy
   ↓
Host approval when required
   ↓
Mission / Trace instrumentation
   ↓
Tool execution
```

This is only the first portion of `orchestrator.rs`. The next study must trace how this safety/protocol layer connects to request construction, streaming, tool follow-up rounds, execution ledger reconciliation, Sentry, fallback providers, context management, and final response generation.
