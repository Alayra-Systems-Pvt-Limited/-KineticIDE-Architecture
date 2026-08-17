# Kinetic Engine — Orchestrator Execution, Planning, and Self-Heal

## Source
`Alayra-Systems-Pvt-Limited/KineticIDE`
`extensions/kinetic-core/kinetic-engine/src/orchestrator.rs`

## Status
**PARTIAL — this pass traced the continuation of the streaming/tool-loop path, blueprint revision, workspace resolution, and legacy planner execution/self-heal logic visible in the studied sections. The remaining tail of the large file still requires line-by-line completion.**

## 1. Mission / Trace envelope implementation

The Orchestrator contains the canonical Rust builders for Mission and Trace pulse bodies shared by the newer agent loop and legacy planner.

Mission steps are limited to visible mutating/terminal actions:

- `write_file`
- `execute_command`
- `delete_file`
- `replace_in_file`
- `move_file`
- `create_directory`
- `propose_edits`

Read/discovery tools and model `emit_pulse` are deliberately excluded from visible execution-step accounting.

The Mission envelope emits an intro and pending steps. File steps can carry a path; command steps are represented as command-type steps.

Trace events carry phase, tool, file, status, narrative, optional command/output, and a real exit code only when the event corresponds to an `execute_command`. The implementation deliberately avoids fabricating forensic fields such as a synthetic heal flag.

Trace status rolls up to:

```text
failed + done → partial
failed only   → error
otherwise     → ok
```

## 2. Structured output hygiene

The engine strips `<think>` and `<reasoning>` blocks before presenting model output to the host. Unclosed reasoning blocks are stripped up to the next JSON object/array boundary when possible.

`pulse_intro_trim()` is sentence-aware and bounded at 220 characters. It avoids terminating on common abbreviations such as `e.g.` and avoids cutting in the middle of a token.

Path policy has a dedicated component-aware check for `.vscode/settings.json`; it does not use fragile suffix matching, preventing false positives such as `my.vscode/settings.json`.

## 3. Blueprint revision

`revise_blueprint()` asks the model to return only sections changed by the user's revision comment.

The revised response is:

```text
original plan + user comment
        ↓
code-mode AI request
        ↓
strip reasoning
        ↓
extract JSON objects
        ↓
truncation check
        ↓
blueprint gate if mutation requires it
        ↓
return revised calls
```

A truncated revision fails closed; it is not re-emitted as a new executable blueprint.

Revision approval preserves the original blueprint's stable `bp_{turn_id}` identity when possible. This is explicitly intended to prevent stale approval IDs between chat and archive representations.

The re-emitted review records:

- original stable ID;
- plan;
- workspace;
- approval requirement;
- file-touch estimate;
- generated narrative;
- user's revision notes;
- `approval_resolution: user_revised`;
- revision timestamp;
- user actor metadata for non-blocking revisions.

Approval is sent through `blueprint_pulse_review` with the dedicated longer blueprint-gate timeout.

## 4. Streaming Rust-owned tool loop

When Rust owns the streaming tool loop, the engine parses `__STRUCTURED_TOOL_CALLS__` into three meaningful classes:

1. no structured sentinel;
2. complete sentinel;
3. incomplete/truncated sentinel, optionally with salvageable calls.

A truncated sentinel with zero salvageable calls produces a structured `tool_calls_truncated` error and a clean user-facing message. The malformed sentinel is never rendered as chat prose and no partial action is claimed.

If calls are salvageable, the engine records that partial salvage and dispatches the safe calls.

After executing initial tool calls, the engine builds OpenAI-style messages:

```text
system
history
user
assistant(tool_calls)
tool(result)
...
```

It then performs bounded follow-up model rounds. Ask-response may receive tools again; other lanes do not automatically receive another tool schema in this follow-up path. The bounded round count is a loop-safety control.

## 5. Plain-text defense

Even when Rust does not own the structured loop — for example, legacy TypeScript sentinel mode or an Anthropic-native endpoint — the Orchestrator records the boundary rather than pretending Rust executed the tools.

On the final plain-text path it strips structured-tool markers defensively before returning the response.

Thus the engine has two separate responsibilities:

- execute structured calls when Rust owns the lane;
- prevent internal protocol markers from leaking into user-visible chat when another transport owns execution.

## 6. Empty-workspace resolution

`resolve_empty_workspace()` handles requests where the active workspace is empty, `.`, or points to a non-existent location.

It selects a default parent:

- Windows: Documents/Kinetic-IDE;
- Unix-like systems: `~/Documents/Kinetic-IDE` when Documents exists, otherwise `~/.kinetic/workspace`.

It derives a concise project folder from a tool hint when possible, creates the target directory, emits an `open_workspace` custom payload, and updates the active workspace.

This is a product-level bootstrap behavior: the agent can establish a workspace before command execution rather than requiring an already-open project.

## 7. Legacy planner markdown rescue

The legacy planner contains a deterministic rescue path when a model response contains Markdown code fences instead of the expected structured tool-call JSON.

It scans fenced blocks and attempts to recover:

```json
{"tool":"write_file","path":"...","content":"..."}
```

The implementation accepts an explicit `path=` fence hint or a clearly filename-shaped preceding line. It deliberately does **not** invent a phantom filename when no explicit path can be established.

Unnamed blocks are skipped and counted for diagnostics.

This is a compatibility layer for models that produce code blocks instead of native structured calls.

## 8. Deterministic bounded retry

If the planner receives either:

- a truncated structured sentinel; or
- no usable tool-call shape;

it performs exactly one deterministic retry with a tightened instruction requiring a complete JSON array of native `write_file` / `execute_command` calls.

The retry is bounded to one attempt rather than recursively retrying indefinitely.

The retry result is stripped of reasoning and structured sentinel markers before downstream parsing.

## 9. Legacy `execute_command` path

The legacy planner recognizes both nested OpenAI-style tool calls and flat `{name, arguments}` calls.

For `execute_command`, it:

1. resolves an empty workspace if necessary;
2. determines the command working directory;
3. enters a maximum three-attempt execution loop;
4. emits progress and health updates;
5. calls the MCP `execute_command` custom request;
6. normalizes canonical `output`/`error` and legacy MCP `content[].text` into one text result;
7. detects `[EXECUTION FATAL]` as the execution-failure marker.

## 10. Autonomous command self-heal

On `[EXECUTION FATAL]`, the planner enters an explicit auto-heal loop.

The observed sequence is:

```text
execute command
      ↓
[EXECUTION FATAL]?
      │
      ├── no → health 100 → launch reveal → finish
      │
      └── yes
            ↓
        health 20
            ↓
        1s UI synchronization pause
            ↓
        update debug state
            ↓
        attempts < 3?
            ↓
        inspect most recently modified file
            ↓
        construct SYSTEM INTERRUPT
            ↓
        record correction in feedback pillar
            ↓
        ask model for corrective tool calls
            ↓
        parse write_file / execute_command
            ↓
        retry
```

The auto-heal prompt explicitly tells the model to fix only the specific failure rather than rewrite the entire file.

The model's corrective response can produce:

- `write_file` → apply the targeted file correction, then retry the original command;
- `execute_command` → replace the failing command/cwd for the next attempt.

If a corrective action cannot be parsed, the loop stops rather than guessing.

The maximum is three attempts, preventing an unbounded autonomous repair loop.

## 11. Debug and feedback integration

Every failed command can update the debug state with the current command, cwd, and output.

Auto-heal records a correction through `feedback_signal::record_correction()` with:

- active workspace;
- observed failure output;
- interrupt prompt;
- correction type `auto_heal`;
- current session ID.

This means self-healing is not merely an isolated retry mechanism: it feeds the engine's feedback/persistent-learning layer.

## 12. Post-command conversational handoff

After command execution, the planner emits an `agent_status_update` completion signal.

It then asks the model for a short natural-language summary in `ask` mode. The follow-up explicitly forbids tools, JSON, and tool calls.

This separates:

```text
execution decision → actual command execution → human-readable explanation
```

rather than asking the model to both execute and narrate the final result in one uncontrolled output.

## 13. Discovery tools during planning

The legacy planner also recognizes discovery calls such as:

- `get_tool_registry`
- `list_dir`
- `grep_search`
- `crawl_website`
- `take_screenshot`

These are executed during planning, and their results are fed back into a subsequent planning step so that the model can produce concrete mutation calls.

This is especially relevant to Vision mode, where URL crawling and screenshots can supply context before file generation.

## 14. Important distinction: two execution generations

The Orchestrator currently contains both:

### Newer Rust-owned agent loop

```text
stream
 → structured tool sentinel
 → Rust parser
 → ToolRouter/dispatch
 → tool results
 → bounded provider follow-up
```

### Legacy planner path

```text
planning response
 → JSON extraction / markdown rescue
 → deterministic retry if necessary
 → direct MCP tool execution
 → command auto-heal
 → natural-language follow-up
```

The source contains explicit feature flags and migration comments around this boundary. This is a compatibility/migration architecture, not evidence that both paths are identical.

## 15. Remaining orchestrator work

The complete `orchestrator.rs` study still requires:

- finishing the remaining `execute_plan()` branches;
- complete discovery-tool handling;
- all write/edit/delete/move/create-directory dispatches;
- ledger/planned-vs-completed accounting at each mutation;
- Blueprint execution and approval handoff;
- Mission/Trace finalization;
- reset/trust/interruption transitions;
- end-of-turn context persistence;
- remaining tests/helpers and final module cross-reference.

Therefore the Orchestrator remains **PARTIAL** until these are traced directly.
