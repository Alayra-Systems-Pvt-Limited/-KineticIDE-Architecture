# Kinetic Engine — Pillars and Tool Implementation Boundaries

## Status
PARTIAL. The current `pillars/` implementation and remaining tool module files were inspected. This establishes the actual persistence, safety, feedback, memory, context, risk, and session-state behavior. Engine completion still requires final cross-reference against orchestrator wiring, intent, MCP dispatch, tests, configuration/assets, and the remaining root modules.

## Source
- `extensions/kinetic-core/kinetic-engine/src/pillars/`
- `extensions/kinetic-core/kinetic-engine/src/tools/`

## 1. Pillar module inventory

The current pillar module exposes:

- `context_manager`
- `feedback_signal`
- `memory_bridge`
- `risk_engine`
- `safety_gate`
- `state_tracker`

`pillars/mod.rs` is only a module declaration surface; the behavior resides in these files.

## 2. Session Context pillar

`context_manager.rs` defines `SessionContext` with four pieces of in-memory state:

```text
files_in_scope
recently_written
recently_read
active_file
```

`add_written()` records a file as both recently written and in scope, without duplicate entries.

`add_read()`, `set_active()`, and `clear()` are explicitly marked as reserved/scaffolding APIs because the comments state that Session Context is **not yet wired into the orchestrator**.

`get_context_summary()` produces a compact prompt-oriented summary:

- active file;
- up to five recent writes;
- up to ten in-scope files.

### Semantic neighbor helper

The same module contains a filesystem-only semantic-neighbor heuristic. It does not use LSP or Tree-Sitter.

For a target file it considers:

1. same-directory source files with recognized extensions;
2. workspace-root entries whose names share a normalized stem after stripping terms such as `service`, `controller`, `model`, `provider`, and `handler`.

The result is capped at the requested maximum.

`build_neighbor_context()` then reads up to five neighbors and includes the first 30 lines of each, bounded by `max_chars`.

This is a lightweight context expansion mechanism rather than a semantic code graph.

## 3. Feedback signal persistence

`feedback_signal.rs` persists user feedback to:

```text
<workspace>/.kinetic/feedback.json
```

The data model is:

```text
FeedbackLog
 └── entries[]
      ├── feedback_type
      ├── original
      ├── correction
      ├── context
      ├── timestamp
      └── session_id
```

Types are:

- `Correction`
- `Preference`
- `Rejection`
- `Approval`

Current public recording helpers explicitly implement **Correction** and **Preference**. `Rejection` and `Approval` exist in the enum but do not have corresponding record helpers in this file.

Corrections are capped at the last 100 total entries. Preferences are deduplicated by the preference text.

`build_feedback_context()` prioritizes preferences and recent corrections and formats them inside `<feedback>` tags for prompt injection.

Therefore this is a local, bounded preference/correction memory mechanism, not model training.

## 4. Cross-session memory bridge

`memory_bridge.rs` persists durable session facts at:

```text
<workspace>/.kinetic/memory.json
```

Session memory contains:

- facts;
- last session ID;
- last update timestamp;
- workspace.

Each `MemoryFact` records fact, source, timestamp, and session ID.

Facts are deduplicated by exact fact text and capped at 50.

`build_memory_context()` emits the newest requested number of facts inside `<memory>` tags.

### Trace distillation

`distill_from_traces()` scans:

```text
<workspace>/.kinetic/pulses/
```

for Markdown files beginning `TRACE_`, sorts them by modification time, reads the three newest, and extracts lines beginning with:

- `Created`
- `Modified`
- `Fixed`
- `Installed`
- `Built`

Those extracted lines are then added to memory with source `trace_distillation`.

This is deterministic textual distillation, not an LLM-based summarizer.

## 5. Risk engine versus command-risk classifier

There are **two distinct risk mechanisms** in the engine.

### `pillars::risk_engine`

`risk_engine.rs` is explicitly annotated as future infrastructure for Sentry tiers. The comment states that Phase 20 uses `tools::command_risk` for `execute_command`.

It defines:

```text
LOW
MEDIUM
HIGH
```

and evaluates:

1. external path access;
2. destructive command signatures;
3. safe development command signatures;
4. ordinary workspace modification;
5. generic fallback.

Because it is `allow(dead_code)` and explicitly described as future Sentry-tier infrastructure, it must **not** be represented as the active command-execution enforcement path.

### Active path

The current active command classifier is `tools::command_risk`, documented in `07_TOOL_SUBSYSTEM_AND_SAFETY.md`.

This distinction is important when changing the security architecture later.

## 6. Deterministic safety gate

`safety_gate.rs` is a hardcoded final safety layer used directly by filesystem mutation and command execution helpers.

### Commands

A blocked-command list contains destructive patterns such as:

- recursive root deletion;
- drive formatting;
- `mkfs`;
- destructive disk writes;
- recursive system permission changes;
- fork bombs;
- destructive sudo deletion.

Matching is case-insensitive substring matching.

### Protected paths

Writes are blocked for sensitive targets including:

- `.env` variants;
- SSH private-key locations;
- `/etc/passwd` and `/etc/shadow`;
- Windows system directories.

### Deletion policy

Deletion is **blocked by default**.

Only paths containing one of the following patterns are automatically allowed:

```text
.kinetic/temp/
node_modules/.cache/
.next/cache/
```

Everything else returns a blocked verdict saying explicit user approval is required.

This means the tool-level `delete_path()` cannot arbitrarily delete normal workspace files through its current safety-gate path.

## 7. Session state tracker

`state_tracker.rs` persists execution state at:

```text
<workspace>/.kinetic/debug_session.json
```

The state model tracks:

- session ID;
- workspace;
- files created;
- files modified;
- commands run;
- passed commands;
- failed commands;
- current step;
- total steps;
- last update.

It provides CRUD-like helper functions for recording file and command events and progress.

### Important overlap

`tools::update_debug_state()` writes to the **same filename** (`.kinetic/debug_session.json`) but uses a different JSON shape keyed by `cwd::command`.

`state_tracker::write_state()` expects a serialized `SessionState` shape.

This is a significant implementation detail: the repository currently contains **two writers/readers targeting the same persistence path with different schemas**. Before changing this subsystem, the actual call graph must be audited to establish which writer is active in each path and whether the two representations can collide.

## 8. Remaining tool module boundaries

The current `tools/` directory also contains three explicit stubs:

### `browser.rs`

Contains only a future implementation note for Playwright-driven browser automation.

**Status: NOT IMPLEMENTED.**

### `sandbox.rs`

Contains only a future implementation note for execution sandbox / VM SDK integration.

**Status: NOT IMPLEMENTED.**

### `pulse_tools.rs`

Contains only a future implementation note for Pulse emission helpers and says it is not wired into the tool registry.

**Status: NOT IMPLEMENTED / NOT REGISTERED.**

These files must not be described as functional browser automation, execution sandboxing, or tool-level Pulse emission.

## 9. Tool implementation surface confirmed

The primary `tools/mod.rs` implementation additionally contains concrete helpers for:

- single file read;
- metadata-only read;
- multiple-file read with per-file errors;
- file write with safety gate + MCP permission;
- mock `web_search`;
- HTTP website crawling;
- configurable screenshot API call / text fallback;
- Python local HTTP serving;
- command execution with safety gate + MCP permission;
- debug-state persistence;
- directory listing JSON;
- regex file search with `.gitignore` support;
- exact-one-occurrence file replacement;
- recursive directory creation;
- cross-filesystem move fallback via copy/remove;
- safety-gated deletion;
- screenshot-summary persistence;
- URL snapshot;
- basic remote HTTP `ui_interact` behavior;
- directory listing text;
- pure-Rust grep search.

The important distinction is that some names imply richer capabilities than the implementation currently provides. For example, `ui_interact` performs HTTP GET/POST behavior and explicitly states that it does **not** automate VS Code UI or provide a browser automation runtime.

## 10. Current tool security chain

For mutation/command paths the verified architecture is:

```text
Model-selected tool
      ↓
Catalog / lane exposure
      ↓
Mode policy
      ↓
Orchestrator / dispatcher
      ↓
Tool implementation
      ↓
Deterministic safety gate
      ↓
MCP permission request
      ↓
Filesystem / process / network operation
```

The exact dispatcher path for every catalog executor still needs final enumeration.

## 11. Engine implementation truth table

| Capability | Current state |
|---|---|
| File reading | Implemented |
| File writing | Implemented + safety gate + MCP permission |
| File deletion | Implemented but safety-gated and blocked by default |
| Command execution | Implemented + safety gate + MCP permission |
| Command risk classification | Implemented in `tools::command_risk` |
| Future Sentry risk engine | Present but currently unused/dead-code guarded |
| Browser automation | Stub |
| Execution sandbox/VM | Stub |
| Pulse tool helpers | Stub |
| URL text crawling | Implemented |
| Pixel screenshot | External API only when configured; otherwise text fallback |
| UI interaction | Basic remote HTTP behavior, not browser automation |
| Session context API | Partially scaffolded/not fully wired |
| Feedback persistence | Implemented |
| Cross-session memory | Implemented |
| Trace distillation | Implemented deterministic extraction |
| Session state tracker | Implemented, but shares path with another state representation |
| Semantic neighbor heuristic | Implemented filesystem heuristic |

## 12. Architectural implications

The engine currently has a strong **policy-and-safety perimeter** around host mutation, but several planned capabilities remain explicitly skeletal.

The most important follow-up is not to add features yet. It is to finish tracing the call graph so we know which of these layers are actually reachable from a model request:

```text
MCP request
 → orchestrator
 → intent / lane
 → catalog
 → policy
 → executor
 → safety gate
 → permission
 → host effect
 → ledger/state/pulse
```

That end-to-end trace is required before Engine completion.
