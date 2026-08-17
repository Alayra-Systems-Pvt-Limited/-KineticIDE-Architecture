# Kinetic Engine — AI, Execution, Tools & Runtime Support

## Study status
**PARTIAL.** This pass traced the AI transport/client, code-execution agent loop, model capability gates, execution ledger, GPU detector, progress channel, and engine source inventory. The remaining major engine areas are the large `orchestrator.rs`, `mcp.rs` full surface, `intent/`, `pillars/`, `tools/` internals, and their end-to-end integration.

## 1. Engine source inventory checkpoint

The current `src/` tree contains the following production areas:

- `ai_client.rs`
- `audio.rs`
- `bridge.rs`
- `code_exec_agent.rs`
- `embedder.rs`
- `execution_ledger.rs`
- `file_enumerator.rs`
- `file_hasher.rs`
- `gpu_detector.rs`
- `graph_retriever.rs`
- `indexer/`
- `infrastructure_master.rs`
- `intent/`
- `main.rs`
- `mcp.rs`
- `model_capabilities.rs`
- `name_index.rs`
- `orchestrator.rs`
- `pillars/`
- `progress.rs`
- `query_engine.rs`
- `regression_guardrails.rs`
- `symbol_extractor.rs`
- `symbol_graph.rs`
- `tools/`
- `vector_engine.rs`
- `workspace_indexer.rs`

Two root modules are explicit stubs: `bridge.rs` says the real bridge is `mcp.rs`; `infrastructure_master.rs` is a future supervisor/infrastructure placeholder. These must not be mistaken for implemented functionality.

## 2. Universal AI client

`ai_client.rs` is the provider-facing HTTP abstraction used by orchestration. It is intentionally OpenAI-compatible at the request boundary but contains explicit handling for native Anthropic Messages API behavior.

### Core contracts

`UniversalClient` owns a reusable `reqwest::Client`. Client-level timeout is disabled; request timeout is applied per request so orchestrator lane policy can choose different time budgets without recreating the HTTP client.

`UnifiedRequest` carries:

- base URL
- API key
- model ID
- prompt/system prompt
- mode
- optional audio/images
- optional history
- optional tools
- request timeout
- manifest-aware output token cap

`AgentChatCompletionRequest` is the multi-turn agent-loop form and carries the complete `messages` array plus optional native tools and per-turn timeout/output limits.

### History safety

Before provider submission, history can be normalized with `normalize_chat_turn` / `sanitize_history_turns`:

- accepted roles: user, assistant/ai, system;
- `ai` is normalized to `assistant`;
- empty/unknown turns are dropped;
- `content`, then `text`, then `value` can supply the body.

A rough token estimator uses serialized message characters at 3.5 characters/token. `trim_messages_for_budget` removes oldest non-system history turns while preserving the first two messages, using the model context window minus output reserve and a 256-token margin.

This is an approximate preflight guard, not a provider tokenizer.

### Error normalization

`AiRequestError` distinguishes provider rate limiting from generic failures. `429` handling extracts `Retry-After` and defaults to 30 seconds. This prevents raw provider JSON from being surfaced directly in UI-facing rate-limit messages.

Context-length failures are recognized by known error strings so the agent loop can retry with reduced history.

### Provider identification

`provider_display_name` maps known endpoint substrings to human-readable labels such as OpenAI, Anthropic, Groq, OpenRouter, Gemini, Tavily and Hugging Face, then falls back to the host portion.

### Anthropic native compatibility

The client recognizes `api.anthropic.com` as a native Messages endpoint and can translate OpenAI-style tool definitions:

```text
function.name
function.description
function.parameters
        ↓
name
 description
 input_schema
```

Anthropic `tool_use` response blocks are converted back into an OpenAI-shaped structured tool-call representation.

## 3. Structured tool-call transport contract

The engine uses a sentinel protocol for providers/paths where structured calls need to be represented as text:

```text
__STRUCTURED_TOOL_CALLS__
<JSON tool-call array>
__STRUCTURED_TOOL_CALLS_END__
```

`TOOL_CALLS_END_MARKER` was added as an additive boundary so streaming consumers can distinguish a complete JSON payload from a stream that died mid-payload. Consumers that do not understand the marker can retain the older sentinel behavior.

`THINKING_SIGNAL` is also defined as `[THINKING_TOKEN]` for the thinking/progress channel.

## 4. Tool capability heuristic

`supports_tool_calling(model_id)` defaults unknown models to tool-capable. A denylist covers known reasoning/model IDs that historically mishandle OpenAI-style tools. An environment variable `KINETIC_FORCE_TOOLS_MODEL_IDS` provides comma-separated model-ID substrings that override the denylist.

This is deliberately a compatibility heuristic rather than a provider capability discovery protocol.

## 5. Multimodal model gate

`model_capabilities.rs` implements a conservative OpenAI-compatible multimodal gate.

Images/audio are rejected for:

- DeepSeek models unless their ID indicates VL/vision;
- `o1-mini`;
- `o1-preview`.

When unsupported multimodal input is detected, the engine strips the attachments and emits a user-visible progress note through `McpBridgeClient` instead of sending a request likely to receive HTTP 400.

This is model-ID based and therefore intentionally heuristic.

## 6. Code execution agent loop

`code_exec_agent.rs` implements the Phase F-1 multi-turn agent execution path.

The runtime is **enabled by default**. `KINETIC_AGENT_LOOP` can explicitly disable it using values such as `0`, `false`, `off`, `no`, `disable`, or `disabled`, providing a rollback switch to the legacy JSON planner.

### High-level lifecycle

```text
User instruction
      ↓
Build seed messages
      ↓
Plan phase
      ↓
Read/search tools only
      ↓
Model returns tool calls / plan JSON
      ↓
Extract complete execution plan
      ↓
Blueprint gate
      ↓
Approved execution
      ↓
Deterministic tool dispatch
      ↓
Receipts / ledger
      ↓
Trace + final summary
```

The plan phase receives tools generated by the engine's lane-aware tool schema builder. The prompt explicitly constrains this phase to read-only inspection and requests a complete execution plan rather than allowing writes during exploration.

The loop supports both native OpenAI-compatible structured tool calls and a legacy JSON-array plan representation. It can perform multiple plan iterations, bounded by `MAX_PLAN_PHASE_ITERATIONS = 15`.

### Context and reliability handling

Before each agent call:

- dynamic timeout is resolved through the orchestrator;
- output token limit is obtained from the orchestrator's manifest-aware budget;
- messages are cloned and trimmed against the model context window.

If a context-length error occurs, the loop performs one controlled retry after removing older history.

If the provider rate-limits the request, the engine emits a structured rate-limit payload to the bridge and can use configured fallback AI credentials when available.

The loop supports interrupt handling and sends stream-end state before returning an interruption result.

### Workspace mutation boundary

Tool dispatch occurs through `Orchestrator::dispatch_openai_tool_for_agent_loop`. This is important: the agent loop itself is not the filesystem implementation. It is the control loop around the orchestrator's tool policy/dispatch layer.

The code imports blueprint approval helpers, trace/forensic logging, plan-call parsing, execution receipts and fallback credentials from the orchestrator. Therefore the eventual full understanding of Code mode requires studying the orchestrator's corresponding functions together with `tools/` policy.

## 7. Execution ledger

`execution_ledger.rs` provides a shared receipt/narrative layer for both the agent and legacy execution paths.

`completed_step_narrative` deliberately describes completed operations using past tense (`wrote`, `ran`, `deleted`, `read`, `listed`) rather than planning language.

The ledger can count successful writes and detect a contradiction where the model's final summary claims that no writes occurred despite completed write receipts.

When contradiction is detected, `finalize_turn_summary` replaces the model-generated summary with a receipt-grounded summary listing executed and failed steps.

This is an important trust boundary:

```text
Actual tool receipt
      ↓
execution ledger
      ↓
summary consistency check
      ↓
model summary OR clamped factual summary
```

The engine therefore does not blindly trust the model's final narrative about filesystem mutations.

## 8. GPU detection and embedding provider selection

`gpu_detector.rs` is intentionally passive and boot-time only.

Three states are exposed:

- `CudaActive` — NVIDIA runtime evidence + enabled flag + expected CUDA provider available;
- `CudaAvailable` — NVIDIA capability exists but the post-install enable flow is incomplete;
- `CpuOnly` — no supported NVIDIA path.

Detection is designed never to block startup or panic. macOS is explicitly CPU-only for the current implementation; Apple Metal/MPS is identified as future work.

The detector records a boot `GpuInfo` snapshot in a `OnceLock` so MCP can publish the state after the server is listening.

The current ORT version is pinned to `1.20.1`. The expected CUDA provider DLL hashes are still placeholders (`PLACEHOLDER_PIN_AT_RELEASE`), which is an implementation-readiness gap for a production GPU distribution.

`embedder.rs` consumes the detector's runtime decision indirectly through `ORT_DYLIB_PATH` and attempts CUDA only when the CUDA feature is compiled and the runtime variable is present; otherwise it uses CPU.

## 9. Progress/event channel

`progress.rs` is the one-way index lifecycle telemetry path:

```text
WorkspaceIndexer
      ↓ progress::emit
bounded Tokio channel
      ↓
MCP forwarding task
      ↓
kinetic_index_progress JSON-RPC notifications
      ↓
TypeScript bridge/UI
```

The channel is bounded at **256**. This is an explicit memory/backpressure protection mechanism.

`IndexProgress` is throttled to roughly one emission per 250 ms per phase. Intermediate progress can be dropped because latest-value semantics make old progress stale.

Lifecycle events (`IndexStarted`, `IndexCompleted`, `SchemaMigration`) are not throttled. If channel backpressure drops them, the engine increments a critical lifecycle-drop counter and logs a critical error.

Telemetry counters expose:

- dropped progress events;
- dropped lifecycle events.

Schema migration is intentionally a lifecycle event because the user must be warned before destructive table replacement.

## 10. Engine safety/quality architecture emerging

Across these modules, the engine has several recurring design patterns:

### Deterministic gates

Model capabilities, tool capability, context budget, execution mode and blueprint approval are checked before the corresponding action.

### Fallbacks

- native provider path → OpenAI-compatible normalization;
- CUDA → CPU;
- native tool calls → structured sentinel/legacy plan parsing;
- primary AI credentials → configured fallback credentials on rate limiting;
- model-generated summary → execution-ledger-clamped summary when contradictory.

### Observability

- progress events;
- rate-limit structured payloads;
- GPU boot telemetry;
- execution receipts;
- forensic prompt/tool logs;
- indexing health counters;
- regression guardrail tests.

### Explicit rollback switches

`KINETIC_AGENT_LOOP` allows reverting from the F-1 agent loop to the legacy planner without a code change.

## 11. Explicitly unimplemented root modules

`bridge.rs` is only a JSON-RPC placeholder. Its own source states that the actual bridge is `mcp.rs`.

`infrastructure_master.rs` is a future supervisor/infrastructure orchestration placeholder.

These should remain documented as stubs rather than inferred architecture.

## 12. Remaining engine study queue

The following are the next required deep dives before the engine can be declared complete:

1. `orchestrator.rs` — complete control plane; this is the largest root module and must be studied in sections and cross-referenced with all imported subsystems.
2. `mcp.rs` — complete RPC surface and TS bridge contract.
3. `tools/mod.rs`, `schemas.rs`, `policy.rs`, `command_risk.rs`, `vision_assets.rs`, `image_generation.rs` and the remaining tool files.
4. `intent/` — classifier and execution-lane routing.
5. `pillars/` — all pillar modules and their lifecycle/state semantics.
6. Remaining `audio.rs` behavior.
7. `file_enumerator.rs` and the full lower-level indexing helpers.
8. Remaining `FileHasher`/`VectorEngine`/`QueryEngine` functions not yet traced.
9. Engine tests/build configuration and the relationship between all guardrails and production code.
10. Final engine end-to-end flow audit, followed by engine completion status.
