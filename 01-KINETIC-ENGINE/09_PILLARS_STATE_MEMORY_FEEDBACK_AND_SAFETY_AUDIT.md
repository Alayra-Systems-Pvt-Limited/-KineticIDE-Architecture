# Kinetic Engine — Pillars: State, Context, Memory, Feedback and Safety Audit

## Status
PARTIAL. This pass completed direct inspection of the six pillar modules. The important result is that several pillar concepts are real local persistence/utilities, while others are explicitly reserved/scaffolding or superseded by newer enforcement code.

## Source
`extensions/kinetic-core/kinetic-engine/src/pillars/`

## 1. Pillar module surface

`pillars/mod.rs` exports:

- `state_tracker`
- `safety_gate`
- `context_manager`
- `memory_bridge`
- `feedback_signal`
- `risk_engine`

The module names represent conceptual pillars, but they do not all have equal runtime maturity.

## 2. State tracker

`state_tracker.rs` defines `SessionState` containing:

- session ID;
- workspace;
- files created;
- files modified;
- commands run;
- passed commands;
- failed commands;
- current step;
- total steps;
- last update timestamp.

Persistence is a JSON file:

```text
<workspace>/.kinetic/debug_session.json
```

Operations are synchronous filesystem reads/writes. The helpers create `.kinetic` when needed and silently fall back to defaults/ignore write failures.

Available operations:

- read state;
- write state;
- record created file;
- record modified file;
- record command result;
- update progress step;
- clear state.

### Important architecture observation

The state tracker is a simple local JSON persistence mechanism. It is **not** the execution ledger. The execution ledger has a richer structured event model and is the authoritative factual execution history identified elsewhere in the engine.

There is also another tool-side debug-state implementation targeting the same `.kinetic/debug_session.json`. This is a potential convergence/ownership issue that must be resolved through final call-graph tracing rather than assuming both APIs are active.

## 3. Safety gate

`safety_gate.rs` provides a compact `SafetyVerdict`:

```text
allowed + reason
```

It contains hardcoded blocked command signatures and protected file-path signatures.

Examples include destructive root deletion, disk formatting, `mkfs`, raw disk writes, broad permission changes, fork bombs, credential files, SSH paths, and Windows system directories.

`check_command()` performs lowercase substring matching against the blocked list.

`check_file_write()` similarly performs lowercase substring matching against protected paths.

`check_file_delete()` is intentionally deny-by-default. Only these path fragments are allowed by this helper:

- `.kinetic/temp/`
- `node_modules/.cache/`
- `.next/cache/`

Everything else returns a blocked verdict requiring explicit user approval.

### Relationship to current command-risk system

This safety gate is an active deterministic primitive used by mutation/command tool paths identified earlier. It is deliberately simple and is **not equivalent to** the newer `tools::command_risk` classifier. The latter provides richer risk levels and shell-wrapper/chained-command analysis.

Therefore the architecture has defense-in-depth rather than one monolithic risk engine.

## 4. Session context manager

`context_manager.rs` defines in-memory `SessionContext`:

- `files_in_scope`;
- `recently_written`;
- `recently_read`;
- optional active file.

`add_written()` is live. `add_read()`, `set_active()`, and `clear()` are explicitly annotated as reserved for future Session Context wiring and are currently dead-code/scaffolding.

`get_context_summary()` produces a compact prompt-ready summary of active/recent files.

### Semantic neighbor heuristic

The same module also implements a filesystem-only neighbor heuristic:

1. collect source files in the active file's directory;
2. derive a normalized stem by removing suffixes such as `service`, `controller`, `model`, `provider`, and `handler`;
3. scan the workspace root for matching names;
4. truncate to a configured maximum.

`build_neighbor_context()` reads up to five neighbors, takes the first 30 lines from each, and stops at the requested character budget.

This is deliberately lightweight and does not require LSP or Tree-sitter.

## 5. Memory bridge

`memory_bridge.rs` provides persistent cross-session memory at:

```text
<workspace>/.kinetic/memory.json
```

A `MemoryFact` stores:

- fact text;
- source;
- timestamp;
- session ID.

`SessionMemory` additionally tracks last session ID, update timestamp, and workspace.

`add_fact()` deduplicates exact fact strings and caps stored facts at **50**.

`build_memory_context()` returns the newest requested facts wrapped in `<memory>` tags for prompt injection.

### Trace distillation

The bridge also reads:

```text
<workspace>/.kinetic/pulses/
```

It searches only Markdown files beginning with `TRACE_`, sorts them by modification time, reads the newest three, and extracts lines beginning with:

- `Created`
- `Modified`
- `Fixed`
- `Installed`
- `Built`

Those extracted statements become memory facts with source `trace_distillation`.

This is therefore a deterministic lexical distillation mechanism, not an LLM summarization pipeline.

## 6. Feedback signal

`feedback_signal.rs` persists user feedback at:

```text
<workspace>/.kinetic/feedback.json
```

Feedback types:

- correction;
- preference;
- rejection;
- approval.

The public persistence operations currently implement correction and preference recording. The log is capped at **100 entries**.

Preferences are deduplicated by exact preference text.

`build_feedback_context()` prioritizes preferences, then recent corrections, and wraps the resulting prompt block in `<feedback>` tags.

Corrections are shortened to 50 characters for prompt injection.

### Runtime significance

This is a genuine local learning signal: user corrections/preferences can persist between sessions and become future prompt context. It is not model training and does not alter provider weights.

## 7. Legacy/future risk engine

`risk_engine.rs` is explicitly annotated `allow(dead_code)` and comments identify it as retained for future Sentry tiers.

Its risk levels are:

- LOW
- MEDIUM
- HIGH

It checks:

1. path outside workspace → HIGH;
2. destructive command signature → HIGH;
3. known safe command signature → LOW;
4. in-workspace modification → MEDIUM;
5. unknown command → MEDIUM.

This module is **not the active Phase 20 execute-command classifier**. Current command execution uses `tools::command_risk` instead.

The architecture documentation must therefore not describe `pillars::risk_engine` as the current command security authority.

## 8. Consolidated persistence map

The engine currently contains several `.kinetic` persistence surfaces:

```text
.kinetic/
├── debug_session.json    ← state tracker / tool debug state
├── memory.json           ← cross-session memory
├── feedback.json         ← user feedback/preferences
├── pulses/               ← trace files consumed by memory distillation
├── artifacts/            ← plan artifacts
└── ...                   ← additional engine/tool/index data
```

The execution ledger is a separate conceptual persistence mechanism and must be mapped independently.

## 9. Key maturity classification

| Component | Current status | Role |
|---|---|---|
| `state_tracker` | Implemented utility | local session/debug state |
| `safety_gate` | Implemented | deterministic hard safety gate |
| `context_manager.SessionContext` | Partial/scaffolded | session context |
| `context_manager` neighbors | Implemented heuristic | semantic-ish filesystem context |
| `memory_bridge` | Implemented | cross-session memory + trace distillation |
| `feedback_signal` | Implemented | corrections/preferences persistence |
| `risk_engine` | Legacy/future | retained Sentry risk abstraction |

## 10. Engine-level architectural conclusion

The pillar system is best understood as a collection of independent safety, continuity, and context primitives rather than one unified runtime subsystem.

The current control-plane hierarchy is approximately:

```text
AI / Orchestrator
      │
      ├── Intent + lane policy
      │
      ├── Tool catalog restrictions
      │
      ├── command_risk classification
      │
      ├── deterministic safety_gate
      │
      └── explicit MCP permission

Continuity/context plane
      │
      ├── state_tracker
      ├── memory_bridge
      ├── feedback_signal
      └── partial SessionContext
```

This separation is important for the final engine architecture because **safety enforcement and conversational continuity are not the same subsystem**.

## 11. Remaining verification required

Before Engine completion, cross-reference must establish:

- every call site of `state_tracker`;
- every call site of `context_manager`;
- every memory/feedback injection point;
- whether trace distillation runs during engine startup in all runtime modes;
- exact ownership of `.kinetic/debug_session.json`;
- all tool catalog executors and their implementations;
- final orchestrator execution lifecycle;
- remaining MCP handlers;
- engine tests/build configuration.

Only after those checks should the Engine status be changed from PARTIAL to COMPLETE.
