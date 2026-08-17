# KineticIDE Architecture Study — Master Ledger

## Purpose
This repository is the durable technical knowledge base for studying the existing KineticIDE implementation. It preserves implementation-level context so the study can be resumed after long gaps in conversation.

## Source of truth
- Source repository: `Alayra-Systems-Pvt-Limited/KineticIDE`
- Source branch studied: `main`
- Source commit observed at study start: `bd8a1b378d7142e0e6e97d6619c3b4ac315bc29c`
- This architecture repository is separate from the product repository.

## Non-modification rule
The KineticIDE source repository is treated as **read-only for this study**. No source file, configuration, documentation, generated artifact, or other byte is to be modified, deleted, renamed, or reformatted as part of the architecture study.

## Study order
1. `extensions/kinetic-core/kinetic-engine` — **STUDY COMPLETE / implementation-status audit recorded**
2. `extensions/kinetic-core` — next
3. `extensions/kinetic-ui` — after core is understood
4. Cross-layer end-to-end audit

## Study method
For every source file we record purpose, public API, dependencies/callers where traceable, state lifecycle, I/O/persistence/network/process boundaries, security controls, tests/guardrails, feature participation, and implementation status.

For significant features we trace:
`UI entry → Core command/event/API → Engine subsystem → external/local dependency → state/persistence → result/error → UI`

## Phase 1 — Kinetic Engine

### Status
**STUDY COMPLETE.**

This means the Engine study phase has been reconciled into the architecture repository. It does **not** mean every Engine capability is production-complete. Implemented, partial, scaffolded, legacy, unused, and known-broken areas are documented rather than conflated with study completeness.

### Durable study records
- `01-KINETIC-ENGINE/01_ENTRYPOINT_AND_RUNTIME.md`
- `01-KINETIC-ENGINE/02_MCP_BRIDGE_AND_DISPATCH.md`
- `01-KINETIC-ENGINE/03_INDEXER_SUBSYSTEM.md`
- `01-KINETIC-ENGINE/04_INDEX_BUILD_RETRIEVAL_STORAGE.md`
- `01-KINETIC-ENGINE/05_EXECUTION_AI_TOOLS_AND_RUNTIME_SUPPORT.md`
- `01-KINETIC-ENGINE/06_ORCHESTRATOR_CONTROL_PLANE_PART1.md`
- `01-KINETIC-ENGINE/07_ORCHESTRATOR_EXECUTION_LIFECYCLE.md`
- `01-KINETIC-ENGINE/08_ORCHESTRATOR_EXECUTION_AND_SELF_HEAL.md`
- `01-KINETIC-ENGINE/09_MCP_RUNTIME_AND_ORCHESTRATOR_REMAINING.md`
- `01-KINETIC-ENGINE/10_TOOLS_POLICY_SCHEMAS_AND_COMMAND_RISK.md`
- `01-KINETIC-ENGINE/11_INTENT_PILLARS_AND_REMAINING_TOOLS.md`
- `01-KINETIC-ENGINE/12_MCP_CONTRACT_AND_ENGINE_CROSS_LAYER_AUDIT.md`
- `01-KINETIC-ENGINE/13_ENGINE_SOURCE_TREE_AND_TEST_AUDIT.md`
- `01-KINETIC-ENGINE/14_BUILD_RUNTIME_ARTIFACTS_AND_FINAL_CLOSURE.md`

### Engine architecture at a glance

```text
                         TypeScript Host / Core
                                  │
                                  │ NDJSON / JSON-RPC-like MCP
                                  ▼
                           ┌─────────────┐
                           │ MCP Server  │
                           └──────┬──────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
              Orchestrator                 Indexer
                    │                           │
          ┌─────────┼─────────┐          ┌──────┴──────┐
          │         │         │          │             │
       Intent     AI Client  Tools   WorkspaceIndexer Retrieval
          │         │         │          │             │
       Lane      Providers   Safety      │        QueryEngine
       Policy      │        Sentry       │        GraphRAG/BM25
          │         │         │          ▼             │
          └──────┬──┴─────────┘       VectorEngine ◄──┘
                 │                         │
          Agent execution             LanceDB/Arrow
                 │                    embeddings/graph
          Execution Ledger
                 │
          streaming / result
                 ▼
             MCP → Core/UI
```

### Major verified responsibilities

**MCP**
- Rust↔host process boundary over stdin/stdout NDJSON.
- Request/response correlation and streaming events.
- Host commands for execution, indexing, retrieval, context lifecycle, feedback, audio, blueprint revision, and health/ping.
- Background index progress forwarding and idle vector compaction.

**Orchestrator**
- Central execution control plane.
- Builds provider requests and mission/trace envelopes.
- Enforces execution lanes and approval policies.
- Supports streaming, structured tool-call recovery, retries/self-healing, discovery, legacy planner paths, and Rust-owned tool loops.
- Maintains session context and execution state.

**Indexer / Retrieval**
- Canonical workspace indexing through `WorkspaceIndexer`.
- Single-flight build coalescing.
- Git-aware incremental behavior.
- Embedding and LanceDB persistence.
- Hybrid vector/BM25 retrieval with RRF.
- GraphRAG expansion and graph health reporting.
- Revision-aware retrieval cache.
- Team HTTP indexer mode with authenticated query/status/index APIs.

**Tools**
- Native workspace/file/search/command/web/serving/debug and related operations.
- Tool schemas expose model-facing capabilities.
- Policy and lane restrictions are applied before execution.
- Command execution has deterministic tiered risk classification.
- Sentry/host approval is an additional permission boundary.
- Destructive command classification includes shell-wrapper recursion, chain splitting, Windows/Unix destructive commands, and force-push protection.

**Intent / Pillars**
- Intent classifier selects an execution lane and capability ceiling.
- Mission builder converts classified intent into execution metadata.
- Persistent context/state/memory/feedback facilities exist as supporting pillars.
- Safety gate exists as a deterministic layer; `tools::command_risk` is the active command-risk authority, while `pillars::risk_engine` is reserved/scaffolded.

## Engine implementation-status principles

The study distinguishes architecture knowledge from production readiness. Examples identified during the study include:

- MCP bridge infrastructure placeholder paths.
- Team `/register` endpoint is a deliberate `501 Not Implemented` contract placeholder for future shard-worker registration.
- Some pillar helpers are currently unused/dead-code according to compiler warnings.
- Historical build/check artifacts in the repository show prior compile failures and warnings; these are recorded as historical artifacts rather than assumed to represent the current source without verification.
- The source contains compatibility/legacy execution paths alongside the newer Rust-owned agent loop.

## Build/verification note

Repository artifacts include historical `cargo check` / build logs. One recorded check reports a LanceDB API visibility mismatch around `FullTextSearchQuery`; another historical check reports `main.rs` workspace/UUID compilation issues. These artifacts are dated evidence, not proof of the current source state. Before production release, the current commit should be built/tested independently.

## Security architecture

```text
Intent / Lane capability ceiling
        ↓
Tool catalog + policy
        ↓
Deterministic command-risk classifier
        ↓
Host/Sentry permission gate
        ↓
OS/filesystem/network operation
```

Blueprint approval and Sentry approval are separate controls. Tool execution is not trusted merely because the model requested a tool.

## Persistence and state

Primary durable/runtime stores and state include LanceDB/Arrow index storage, embedding/vector tables, symbol graph data, file/Git revision metadata, process-local retrieval cache, execution ledger, session/context/memory/feedback facilities, and host-managed persisted session history for rehydration.

## Engine completion boundary

The Engine study is complete enough to move to Core because the architecture repository now contains:

1. source-tree inventory;
2. subsystem responsibilities;
3. principal APIs and transport contracts;
4. state/persistence boundaries;
5. security/safety boundaries;
6. indexing/retrieval architecture;
7. orchestration lifecycle;
8. tool policy and risk model;
9. implementation/stub/legacy distinctions;
10. build/test/regression evidence;
11. explicit cross-layer contracts to Core.

If later source changes are made to KineticIDE, this ledger must be revalidated against the new source commit rather than assumed unchanged.

## Current phase pointer
**Phase 2 — `extensions/kinetic-core`**

Next study target: enumerate the complete Core tree, then trace the Core↔Engine boundary file-by-file without modifying the KineticIDE source repository.
