# Kinetic Engine — Orchestration Execution Authority and Current Repo Reality

## Status
**PARTIAL — the orchestration control plane is now substantially traced, but final closure still requires the complete `execute_plan` body/call graph, MCP dispatch implementation, and final test/build/config audit.**

## 1. Authoritative architecture chain

The repository's own orchestration audit establishes the intended chain:

```text
User message
    ↓
kinetic-core processInstruction / intent wall
    ↓
kinetic-engine execute_plan
    ↓
classify_with_context
    ↓
CLASSIFIER_RESULT
    ↓
ExecutionLane
    ├── AskResponse
    ├── PlanArtifact
    ├── CodeExec
    └── VisionClone
    ↓
AI client + lane-filtered tool catalog
    ↓
stdout NDJSON / stream_token / tool payloads
    ↓
MCPBridge
    ↓
kinetic-ui Dashboard / pulses
```

This is more authoritative than reconstructing architecture from filenames alone because it is explicitly maintained in the repository's orchestration audit.

## 2. Orchestrator is not only an AI loop

`orchestrator.rs` contains several independent control responsibilities:

- streaming tool-call parsing;
- truncated JSON salvage;
- tool-call sentinel handling;
- lane conversion from catalog lane keys;
- health-probe timeout calculation/cache;
- context/output token budgets;
- blueprint approval policy;
- plan mutation-risk analysis;
- Mission/Trace pulse generation;
- tool execution lifecycle;
- execution-ledger grounding;
- cooperative interruption;
- fallback AI credentials;
- session/context state.

Therefore it is the central execution policy layer rather than merely an LLM wrapper.

## 3. Streaming structured-tool protocol

The engine defines a structured tool-call sentinel:

```text
__STRUCTURED_TOOL_CALLS__
```

with an optional end marker.

`classify_sentinel_payload()` has three states:

```text
Complete
Incomplete
None
```

This is an important reliability improvement over all-or-nothing JSON parsing.

### Salvage algorithm

When a stream ends during a JSON array, the engine scans top-level objects while tracking:

- brace depth;
- string state;
- escape state.

It can therefore recover valid leading tool-call objects without being confused by braces inside JSON strings.

Invalid/incomplete trailing objects are not fabricated.

### Rendering defense

`strip_sentinel_markers()` removes the structured-call sentinel/end marker and JSON tail before text is sent to chat rendering. This prevents malformed tool protocol data from becoming user-visible prose.

## 4. Tool-call follow-up ceiling

The orchestrator defines:

```text
STREAMING_TOOL_FOLLOW_UP_MAX_ROUNDS = 8
```

This bounds recursive/iterative tool-follow-up behavior. The engine therefore has an explicit termination ceiling instead of allowing an agent to continue indefinitely.

The exact call sequence for each round must still be traced through the remaining `execute_plan` body and MCP methods before calling the end-to-end loop closed.

## 5. Blueprint approval is a separate policy

The orchestrator computes mutation scope before executing a plan.

It tracks touched paths for tools such as:

- `write_file`
- `propose_edits`
- `replace_in_file`
- `delete_file`
- `create_directory`
- `move_file`
- `capture_screenshot`
- `generate_image`
- `convert_asset`

It separately marks destructive/terminal operations including command execution, deletion, file replacement/move, directory creation, image generation/conversion, and `ui_interact`.

The structural threshold is:

```text
≥ 3 distinct files
OR
≥ 4 operations on one file
OR
any destructive/terminal tool
```

The host-threaded approval policy then has two modes:

### `threshold`
Use structural thresholds; safe small plans may auto-continue.

### `always` (default)
Every mutating plan requires explicit Approve/Reject/Revise.

The blueprint approval policy is intentionally **independent from Sentry approval state**.

## 6. Blueprint narrative is deterministic

`build_blueprint_narrative()` does not make another AI call.

It derives:

- operation count;
- file count;
- deletes;
- commands;
- test/build indicators;
- verification instructions.

This means the approval UI can explain the plan without adding another model round or inventing execution details.

## 7. Mission and Trace are generated from execution facts

The orchestrator has shared builders for Mission and Trace envelopes.

Mission steps are limited to visible mutation/terminal tools:

- `write_file`
- `execute_command`
- `delete_file`
- `replace_in_file`
- `move_file`
- `create_directory`
- `propose_edits`

Read tools and model-generated `emit_pulse` are deliberately not considered Mission steps.

Trace events carry factual fields such as:

- phase;
- tool;
- file;
- status (`done`/`failed`);
- narrative;
- command;
- output;
- real command exit code where available.

The implementation explicitly avoids fabricating an exit code for non-command steps and avoids fabricating a self-heal signal.

This is consistent with the execution-ledger principle: telemetry should describe what actually happened rather than what the model claimed happened.

## 8. Pulse body contract

Mission uses:

```xml
<intro>...</intro>
<step type="..." path="..." state="pending">...</step>
```

Trace uses:

```xml
<intro>...</intro>
<event phase="..." tool="..." file="..." status="..." exit_code="..."></event>
```

XML values are escaped before insertion.

The bridge remains the surrounding XML/pulse transport authority; Rust builds the body according to the shared contract.

## 9. Cooperative cancellation

The orchestrator owns an `AtomicBool` interruption flag. The host can request `interrupt_orchestration` while a streaming operation is running.

This is cooperative cancellation rather than process termination: the engine's streaming/execution path must observe the flag and unwind.

The exact propagation into every tool subprocess remains part of the final call-graph audit.

## 10. Dynamic health timeout

The orchestrator keeps a TTL cache keyed by normalized provider base URL.

The cache records:

- resolved timeout;
- cache timestamp;
- whether the health probe failed.

The purpose is to adapt request timeout behavior to observed provider RTT while suppressing repeated failing probes during a failure backoff period.

This belongs to orchestration policy rather than provider implementation.

## 11. Context and output budget are host-threaded

The orchestrator receives host-provided values for:

- output token cap;
- context window;
- code-lane timeout;
- blueprint gate timeout;
- Sentry gate timeout.

A value of `0` generally represents absent/legacy configuration, preserving prior behavior.

This means IDE-side model manifest/budget knowledge is deliberately passed into Rust instead of duplicated in the engine.

## 12. Repository audit: current implementation vs product intent

The repository's `KINETIC_ORCHESTRATION_AUDIT_AND_ROADMAP.md` is an important source because it explicitly records known gaps.

### Current intended/reality mismatches documented there

**Routing**

Rust classifier + lane ceiling exists, but historical TS routing bypasses and classifier-disabled surfaces require alignment.

**Ask mode**

A historical critical failure existed where tool-call-only streaming could end with no natural follow-up completion, producing an empty assistant bubble. The current orchestrator contains the sentinel classification/salvage infrastructure intended to address this class of failure; final behavior still needs end-to-end verification through current `execute_plan` and bridge code.

**Plan mode**

The audit records historical direct artifact writing and Windows-centric path handling as a gap requiring the phased fixes. The current orchestrator contains explicit blueprint approval/path-touch policy, but final artifact execution behavior must be checked rather than inferred.

**Code mode**

The repository identifies the need for a lifecycle owner around the Rust agent loop, bridge `emit_pulse`, and scattered handlers.

**IPC**

JSON-RPC/NDJSON reliability is an explicit concern, with correlation IDs and stdout scheduling mitigations in Rust MCP and TypeScript `mcpBridge`.

**Streaming UI**

The product depends on cohesion between the stream contract, Ask/tool completion, Code pulses, and UI pulse handlers.

**Auto/BYOK routing**

The audit identifies possible divergence between Traffic Cop/model-pool selection and the Rust classifier. This must be checked during the Core study because the TypeScript side owns significant routing/pool behavior.

## 13. Critical architectural insight

There are **two historical orchestration surfaces** documented in the repo:

```text
Rust agent loop
        ↕
TS stream_end → dispatchCanonicalToolCall
```

The long-term architecture needs one clear authority for tool execution. The repository's own audit calls this out as a root-cause cluster.

This is not yet a claim that the dual surface remains equally active in the current commit; it is a required verification item because the engine cannot be understood correctly without following both sides.

## 14. Current end-to-end model

The most defensible model after this pass is:

```text
                     USER
                       │
                       ▼
              kinetic-core host
                       │
                processInstruction
                       │
                       ▼
                Intent / mode wall
                       │
                       ▼
               Rust Orchestrator
                       │
          ┌────────────┼─────────────┐
          │            │             │
       classify     AI client    context/budget
          │            │             │
          ▼            ▼             │
      Lane ceiling  model stream     │
          │            │             │
          └──────┬─────┘             │
                 ▼                   │
           tool-call protocol ◄──────┘
                 │
                 ▼
          lane-filtered tools
                 │
                 ▼
         policy / risk / gate
                 │
                 ▼
          host-side effect
                 │
          ┌──────┴──────┐
          ▼             ▼
       Ledger        Mission/Trace
          │             │
          └──────┬──────┘
                 ▼
                MCP
                 │
                 ▼
             kinetic-ui
```

This model is still provisional at the final dispatch edge until the remaining MCP and Core bridge code is traced.

## 15. Remaining engine closure checklist

- [ ] Complete every `execute_plan` branch.
- [ ] Map every catalog tool name → exact Rust/TS executor.
- [ ] Verify timeout enforcement against catalog declarations.
- [ ] Trace `McpBridgeClient` request/response/cancellation lifecycle.
- [ ] Trace stdout NDJSON ownership and stream-end ownership.
- [ ] Verify current TS canonical dispatch is still active and where.
- [ ] Audit regression guardrails/tests.
- [ ] Audit Cargo features/dependencies/build scripts.
- [ ] Audit engine-level environment/configuration contracts.
- [ ] Reconcile all documented historical gaps with current source.
- [ ] Mark engine COMPLETE only after the above are verified.
