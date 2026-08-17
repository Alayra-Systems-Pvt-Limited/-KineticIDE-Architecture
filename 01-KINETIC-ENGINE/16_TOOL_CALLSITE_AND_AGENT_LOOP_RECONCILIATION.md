# Kinetic Engine — Tool Call-Site and Agent-Loop Reconciliation

## Status
**FINAL CLOSURE IN PROGRESS.**

This pass traces the concrete tool dispatch in `orchestrator.rs` rather than relying on the tool catalog alone.

## Legacy / structured planning execution path

The structured tool-call execution branch parses both common shapes:

```text
{"function":{"name":"...","arguments":"..."}}
```

and:

```text
{"name":"...","arguments":...}
```

It normalizes arguments before routing.

The legacy execution dispatcher explicitly handles or delegates:

- `emit_pulse`
- `read_file`
- `read_multiple_files`
- `list_directory` / `list_dir`
- `search_files` / `grep_search`
- `get_editor_state` → TypeScript/MCP
- `get_diagnostics` → TypeScript/MCP
- `web_fetch` → TypeScript/MCP
- `replace_in_file`
- `create_directory`
- `move_file`
- `delete_file`
- `capture_screenshot`
- `snapshot_ui`
- `ui_interact` where wired in the remaining branch
- `web_search`
- `get_tool_registry`
- `crawl_website`
- `take_screenshot`
- `launch_browser` → host/UI open-browser payload
- `generate_image`
- `convert_asset`
- command execution through the autonomous command path
- `propose_edits`

The exact implementation is intentionally split between native Rust tools and host/MCP calls.

## Command execution path

The newer Rust agent-loop path performs deterministic command-risk classification before host execution:

```text
agent tool call
    ↓
classify_command_risk(command)
    ├── Blocked → reject
    ├── AutoApprove → LOW
    ├── Medium → MEDIUM
    └── High → HIGH
    ↓
if session not trusted
    ↓
show_sentry_review
    ↓
user allow / reject
    ↓
MCP execute_command
    ↓
bridge result
    ↓
state_tracker record_command
```

This confirms that the command-risk layer is active for the Rust-owned agent loop, not merely a dormant helper.

## Write path

The agent-loop `write_file` path does not directly call `Tools::write_file`. Instead it:

1. normalizes the target path relative to the active workspace;
2. applies `validate_workspace_relative_write` for the current lane;
3. emits agent file status;
4. sends `propose_edits` to the TypeScript host;
5. checks the returned `success` flag;
6. records created/modified state through `state_tracker`;
7. records the written path in session context.

Therefore there are two distinct write mechanisms:

```text
Legacy structured tool execution
    → Tools::write_file / Sentry + safety gate

Rust agent loop
    → lane path policy
    → MCP propose_edits
    → TypeScript write gate
```

This distinction is architecturally important and must be preserved in the final feature/security map.

## Mutation tool approval paths

For the legacy structured execution path:

- `replace_in_file` → explicit HIGH Sentry review → native replacement primitive.
- `create_directory` → MEDIUM Sentry review → native directory creation.
- `move_file` → MEDIUM Sentry review → native move/copy primitive.
- `delete_file` → HIGH Sentry review → `Tools::delete_path` → local safety gate.
- `propose_edits` → host `propose_edits` request.

For command execution, the autonomous command branch uses its own retry/self-heal flow and communicates through the host execution endpoint.

## Asset tool lane restrictions

`generate_image` and `convert_asset` are checked by `is_asset_tool_allowed_in_lane` before execution. The legacy structured path performs the actual generation/conversion.

The Rust agent-loop dispatcher explicitly returns a message that these tools require the legacy execution pipeline rather than executing them directly. Thus:

```text
Rust agent loop → asset tool request → lane check → legacy-pipeline requirement
Legacy structured loop → asset tool request → actual asset operation
```

This is a deliberate compatibility boundary, not evidence that the asset tools are absent.

## Discovery / planning tools

Before final execution, the orchestrator can execute discovery tools and feed their results back into planning. Confirmed discovery examples include:

- `get_tool_registry`
- `list_dir`
- `grep_search`
- `crawl_website`
- `take_screenshot`

This supports a two-stage pattern:

```text
instruction
  ↓
planning
  ↓
discovery tool
  ↓
result injected into context
  ↓
re-plan
  ↓
actual mutation/execution tools
```

## Auto-heal command path

The legacy autonomous command path:

1. executes a command;
2. detects `[EXECUTION FATAL]`;
3. records debug state;
4. identifies a recently modified candidate file;
5. asks the model for a targeted repair;
6. permits `write_file` or a replacement command;
7. retries up to three attempts;
8. produces a natural-language summary constrained by the actual output.

This is separate from the simpler Rust agent-loop command dispatcher.

## Execution ledger integration

The orchestrator collects completed tool narratives rather than merely planned steps. The final summary is generated from the completed ledger and is clamped when model output contradicts the recorded execution.

This closes an important trust boundary:

```text
model plan ≠ completed action
completed ledger = authoritative action record
```

## Tests observed in orchestrator

The source contains colocated tests covering at least:

- JSON tool-object extraction;
- NDJSON/markdown tool-call parsing;
- truncation handling;
- sentinel splitting;
- Ask-mode tool-summary fallback;
- structured tool-call recovery behavior.

These tests are part of the engine regression surface and must remain in the final audit.

## Remaining closure work

The main unresolved item is now **coverage verification**, not discovery of another major tool architecture:

- compare the model-facing catalog/schema names with every dispatcher branch;
- identify catalog tools that intentionally terminate at the TypeScript host;
- identify any catalog tool with no dispatcher branch;
- verify every mutating branch has the intended policy/safety/Sentry layer;
- reconcile the legacy loop and Rust agent-loop tool sets;
- record intentional differences as compatibility architecture.

After this reconciliation, the remaining Engine work is the final security + end-to-end audit and completion gate.
