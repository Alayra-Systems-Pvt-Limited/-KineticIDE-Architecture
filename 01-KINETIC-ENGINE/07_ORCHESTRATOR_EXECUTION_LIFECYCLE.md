# Kinetic Engine — Orchestrator Execution Lifecycle

## Source
`Alayra-Systems-Pvt-Limited/KineticIDE`
`extensions/kinetic-core/kinetic-engine/src/orchestrator.rs`

## Status
**PARTIAL — request/dispatch/streaming/turn lifecycle traced so far. The remaining execute_plan branches and legacy planner/tool execution paths still require line-by-line study.**

## 1. Runtime state
`Orchestrator::new()` establishes bounded conversation context, `UniversalClient`, `SessionContext`, session-distillation/trust state, health/timeout cache, interrupt flag, host code-lane timeout, Blueprint approval policy (`always` by default), output token cap, and context-window fields. The Orchestrator is therefore a session/control coordinator, not merely an HTTP client.

## 2. Dynamic provider timeout
`resolve_dynamic_timeout()` chooses lane-specific base timeout and caches provider health by normalized base URL. Normal probe cache is 30 seconds; failed probes back off for 90 seconds. RTT below two seconds uses base timeout; slower probes add 60 seconds; failures use base timeout while suppressing repeated probes during backoff. Health is checked through `UniversalClient::check_health()`.

## 3. Non-streaming request pipeline
```text
user prompt
 → session context summary
 → mode-specific system prompt
 → neighbor context
 → memory context
 → feedback context
 → recent interaction history
 → dynamic timeout
 → lane-specific tool schemas
 → UnifiedRequest
 → UniversalClient
```

System prompts explicitly distinguish Ask, Vision, Plan, and Code behavior. Plan mode is planning-only and does not receive execution tools; Code/Vision use native tool calls and workspace-root restrictions.

## 4. Provider request and fallback
`UnifiedRequest` carries provider credentials/model, final prompt/system prompt, mode, multimodal inputs, history, lane tools, timeout, and optional output-token cap. Rate limits emit a structured `rate_limited` event. If fallback credentials exist, the request is retried against fallback URL/key/model; otherwise a structured `stream_error` is emitted.

## 5. Streaming architecture
`send_prompt_streaming()` accumulates the complete model response while forwarding tokens through an asynchronous channel to MCP/UI. The interrupt flag suppresses forwarding after cancellation. The full accumulation is retained because the engine may need to inspect it for structured tool-call sentinels after streaming.

```text
UniversalClient stream
 → token callback
 → async channel
 → forwarding task
    ├─ append to accumulated response
    └─ send token to MCP/UI
```

Streaming rate limits can trigger the same fallback mechanism. Cancellation returns the partial accumulated response after stream termination. Other failures are classified into structured 429/401/404/403/ERROR stream errors.

## 6. Rust-owned structured tool loop
Rust owns the structured tool loop when legacy TypeScript sentinel tools are disabled, an explicit tool lane is present, and the endpoint is not Anthropic-native. This is an explicit migration boundary between Rust-native and legacy/endpoint-specific tool execution.

The sentinel parser distinguishes no sentinel, complete calls, zero-call sentinel, incomplete/salvageable calls, and incomplete/unrecoverable calls. An unrecoverable truncated tool-call response fails closed: a `tool_calls_truncated` error is emitted and the malformed sentinel is never rendered as normal prose. Partial salvage is separately logged.

## 7. Follow-up tool rounds
The Rust-owned follow-up consumes OpenAI-style assistant `tool_calls`, dispatches each call through `dispatch_openai_tool_for_agent_loop()`, appends tool result messages, and sends follow-up model requests. The loop has a bounded maximum round count. Final assistant text is emitted to MCP. In Ask mode, if the model returns no summary after tools, the engine can synthesize one from tool payloads or emit an explicit notice.

## 8. `execute_plan` turn coordinator
The traced entry sequence is:

1. clear interrupt;
2. apply host timeout/Blueprint/output/context settings;
3. establish active workspace;
4. clear workspace pillar state;
5. sanitize multimodal inputs for the selected model;
6. restore persistent `session_history.json` when the in-memory context is empty;
7. create a session ID;
8. perform one-time trace distillation into memory;
9. classify intent with the Rust Enterprise Intent Classifier;
10. emit `CLASSIFIER_RESULT`;
11. build/emit missions;
12. handle reset/identity control paths;
13. emit lane-specific progress;
14. dispatch by classifier-selected execution lane.

## 9. Intent classification
`IntentContext` carries user mode, attachment state, and contextual placeholders. The classifier returns task intent, execution lane, confidence, capability cap, and reasons. The engine explicitly treats this Rust decision as the per-turn lane authority rather than allowing dashboard mode alone to decide execution.

## 10. Mission construction
`mission_builder::build_missions()` runs immediately after classification. `MISSION_LIST` is emitted with turn ID, missions, intent, lane, and attachment state. This provides a structured host-visible representation before execution.

## 11. Lane dispatch observed
**AskResponse:** Tavily/HuggingFace/FLUX-like endpoints use non-streaming `send_prompt`; ordinary endpoints use streaming.

**PlanArtifact:** requests structured plan JSON, extracts JSON objects, and persists Markdown artifacts beneath `.kinetic/artifacts/`; invalid JSON becomes an explicit engine error.

**CodeExec:** enters the coding/tool execution machinery and maintains a planned-vs-completed write ledger.

**VisionClone:** supported as a distinct classifier lane with browser/reference and asset-generation behavior in its system prompt.

## 12. Persistent context
When initially empty, `context_buffer` restores up to ten turns from global-storage `session_history.json`. This short session context is distinct from the longer-lived pillar memory/trace mechanisms.

## 13. Layered controls identified
The Orchestrator currently contains multiple separate controls: intent/capability classification, workspace-root path restrictions, health-aware timeouts, provider fallback, structured tool parsing, truncation fail-closed behavior, Blueprint approval, interruption, bounded tool rounds, and execution/trace ledger mechanisms.

## 14. Remaining orchestrator study
Still to trace before this file can be marked complete:

- complete `resolve_streaming_turn_tools_rust_owned()`;
- complete PlanArtifact branch;
- complete CodeExec branch;
- legacy planner execution loop;
- Blueprint persistence/approval/revision paths;
- `dispatch_openai_tool_for_agent_loop()`;
- execution-ledger integration at each mutation;
- context-buffer update/persistence after turns;
- Mission/Trace end-of-turn emission;
- interrupt/reset/trust transitions;
- remaining helpers and tests.

These will be verified against their called modules rather than inferred from the Orchestrator alone.
