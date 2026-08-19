# Kinetic Engine — Final Public-Surface, Caller, and Security Reconciliation

## Status
**FINAL ENGINE STUDY GATE CLOSED.**

This record closes the remaining reconciliation work after the Engine source-tree/test study. It is based on the studied KineticIDE `main` source revision observed throughout the Engine study (`bd8a1b378d7142e0e6e97d6619c3b4ac315bc29c`).

This is an architectural/source audit. It does **not** claim that a fresh `cargo test`, release build, packaged installer, or production deployment test was executed during this study.

## 1. Public Engine surfaces reconciled

### Orchestrator
The principal public/public-within-engine surfaces are now accounted for:

- `Orchestrator::new` — created by the MCP sidecar startup path.
- `signal_interrupt` / interrupt state — used by host/MCP interruption handling and the streaming execution lifecycle.
- `reset_context_buffer` — exposed through the dedicated engine reset RPC and paired with host-side session-history cleanup.
- `rehydrate_context_buffer` — exposed through the dedicated rehydration RPC and used by Core/session reopening.
- `estimate_tool_schema_tokens` / `estimate_context_buffer_tokens` — telemetry surfaces for the context-budget system.
- `compact_replace` — engine half of the dual-store auto-compact transaction; paired with the Core auto-compact module.
- `revise_blueprint` — dedicated Blueprint revision RPC path; Core passes the original Blueprint ID so the revision remains correlated with the original approval record.
- `execute_plan` — primary orchestration entrypoint from the MCP request dispatcher.
- `dispatch_openai_tool_for_agent_loop` — shared tool-dispatch authority called by the Rust agent loop.

The public `FallbackAiCredentials` data contract is consumed by the orchestrator's provider fallback paths.

### MCP bridge
`McpBridgeClient` public methods are accounted for as host-boundary primitives rather than independent business authorities. The important surfaces are:

- permission request / Sentry validation;
- progress and custom payload emission;
- stream token / stream-end lifecycle;
- custom request and timeout-aware custom request;
- streaming-turn correlation via the RAII `StreamingTurnGuard`.

The timeout-aware request is the authoritative gate-capable request primitive; the non-timeout request remains a compatibility wrapper for non-gate operations.

### Tool surface
The `Tools` implementation and catalog were reconciled at the schema/implementation level. The catalog currently declares **20 tools**. Tool schemas are generated from `extensions/kinetic-core/tools/toolCatalog.json`, with lane allowlists and policy filtering applied before model exposure.

Native Rust tool implementations include filesystem reads/search, directory operations, file mutation primitives, command support, URL/content extraction, screenshot fallback, image generation/conversion, UI interaction helpers, debug state, and result normalization. Host-mediated tools include editor state, diagnostics, web fetch, pulse emission, and edit proposal application.

## 2. Caller reconciliation

The major call paths are now explicit:

```text
Kinetic Core / TypeScript host
        ↓ MCP request
McpServer dispatcher
        ↓
Orchestrator::execute_plan / dedicated context & Blueprint RPCs
        ↓
Intent classifier → lane/capability ceiling
        ↓
AI client / Rust agent loop
        ↓
Tool catalog + lane policy
        ↓
Rust-native executor OR McpBridgeClient host-mediated operation
        ↓
Sentry / VS Code / filesystem / process / provider
        ↓
Execution ledger + state + progress/stream
        ↓
MCP result → Core/UI
```

The default Code/Vision execution path may enter `code_exec_agent.rs`; the legacy planner remains a deliberate fallback/compatibility path. Both paths share the Blueprint policy and Mission/Trace envelope contracts.

## 3. Test and guardrail reconciliation

The Engine source study covered the relevant colocated tests and the cross-module `regression_guardrails.rs` suite. The guardrails pin, among other things:

- intent-to-lane routing and capability caps;
- tool catalog parsing and lane restrictions;
- BM25/vector parallel retrieval and RRF fusion;
- singleton embedder initialization;
- bounded indexing/progress behavior and file-size caps;
- schema migration ordering and canonical vector pipeline ownership;
- command-risk shell-wrapper handling and tokenized blocked-command checks;
- streaming sentinel completeness/salvage and no-prose-on-truncation;
- Blueprint/Sentry separation, policy threading, stable Blueprint IDs and fail-closed truncation;
- Mission/Trace schema/correlation and live step-state persistence;
- context-window/output-budget threading, reset/rehydration/auto-compact surfaces;
- Sentry timeout and cancellation cleanup;
- execution-ledger honesty and summary clamping.

The intent golden tests explicitly cover code, ask, plan, vision, attachment, capability-cap, and auto-mode cases. The tool schema tests verify catalog size, lane counts, read-only plan subsets, and forbidden tool exposure.

## 4. Security reconciliation

The effective security architecture is layered rather than delegated to one function:

```text
Surface routing
    ↓
Intent classifier / lane ceiling
    ↓
Tool catalog + lane policy
    ↓
Rust command-risk / deterministic safety checks
    ↓
Host Sentry / JIT approval where required
    ↓
Filesystem / process / network effect
```

Important source-level observations:

- destructive file deletion is denied by the deterministic safety helper unless the path is in its narrow allowed cache/temp set;
- command-risk classification contains shell-wrapper detection and a blocked-command tokenization layer;
- `.vscode/settings.json` and several sensitive paths are explicitly guarded;
- Blueprint approval is intentionally independent from Sentry approval;
- gate RPCs have bounded timeouts and pending-request cleanup;
- streaming turns carry correlation IDs and have stale/interrupt suppression;
- the optional Team indexer server requires an API key configuration;
- MCP sidecar operation is local stdin/stdout rather than an exposed network listener.

### Security hardening items preserved as non-blocking study findings

The source study also identifies places that should be hardened before claiming high-assurance production security:

1. Some low-level `Tools::*` functions are reusable public primitives and do not themselves enforce a workspace-root boundary; the authoritative enforcement is therefore distributed across callers, policy, and host/Sentry paths.
2. `capture_screenshot`, `create_directory`, `move_file`, and related low-level helpers should receive explicit path-boundary tests if they are ever exposed as a standalone SDK surface rather than remaining behind the current orchestration policy.
3. `ui_interact` performs HTTP-level GET/POST behavior and should retain explicit SSRF/network policy at the host boundary when used against arbitrary URLs.
4. The simple `pillars::safety_gate` is a deterministic defense layer, not the complete security model; production claims must consider the stronger command-risk + lane-policy + Sentry chain used by orchestration.

These are **hardening recommendations**, not evidence that the studied architecture lacks security controls.

## 5. Memory/context reality

The Engine already contains three different persistence concepts that must not be conflated:

- rolling session history (`session_history.json` / in-memory `context_buffer`);
- workspace memory facts (`.kinetic/memory.json`);
- feedback and correction history (`.kinetic/feedback.json`).

Trace distillation can promote recent trace facts into workspace memory, while feedback is injected separately. This is an important foundation for the future persistent, local-first memory system, but it is **not yet the full long-lived adaptive memory architecture envisioned for future Kinetic**.

## 6. Engine completion decision

The remaining study gate was reconciliation, not another broad reread of already documented architecture. The source, callers, tests/guardrails, tool catalog, principal execution graph, and security boundary have now been reconciled sufficiently to close the Engine phase.

**ENGINE STUDY: COMPLETE.**

Next active phase:

**`extensions/kinetic-core` — full file-by-file study.**

The source repository remains read-only throughout the architecture study.
