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
1. `extensions/kinetic-core/kinetic-engine` — **COMPLETE**
2. `extensions/kinetic-core` — **ACTIVE NEXT PHASE**
3. `extensions/kinetic-ui` — after Core is understood
4. Cross-layer end-to-end audit

## Study method
For every source file we record purpose, public API, dependencies/callers where traceable, state lifecycle, I/O/persistence/network/process boundaries, security controls, tests/guardrails, feature participation, and implementation status.

For significant features we trace:
`UI entry → Core command/event/API → Engine subsystem → external/local dependency → state/persistence → result/error → UI`

## Phase 1 — Kinetic Engine

### Status
**ENGINE STUDY COMPLETE.**

The Engine source tree, major subsystems, tool catalog, execution paths, tests/guardrails, public surfaces/callers, security boundaries, and principal end-to-end flows have been reconciled sufficiently to close the architecture-study phase. This is an architectural/source-study completion decision, not a claim that every future production-hardening or fresh runtime validation has already been executed.

### Durable study records
- `01-KINETIC-ENGINE/01_ENTRYPOINT_AND_RUNTIME.md`
- `01-KINETIC-ENGINE/02_MCP_BRIDGE_AND_DISPATCH.md`
- `01-KINETIC-ENGINE/03_INDEXER_SUBSYSTEM.md`
- `01-KINETIC-ENGINE/04_INDEX_BUILD_RETRIEVAL_STORAGE.md`
- `01-KINETIC-ENGINE/05_EXECUTION_AI_TOOLS_AND_RUNTIME_SUPPORT.md`
- `01-KINETIC-ENGINE/06_ORCHESTRATOR_CONTROL_PLANE_PART1.md`
- `01-KINETIC-ENGINE/06_ORCHESTRATOR_MCP_INTENT_AND_TOOL_CONTROL.md`
- `01-KINETIC-ENGINE/07_ORCHESTRATOR_EXECUTION_LIFECYCLE.md`
- `01-KINETIC-ENGINE/07_TOOL_SUBSYSTEM_AND_SAFETY.md`
- `01-KINETIC-ENGINE/08_ORCHESTRATOR_EXECUTION_AND_SELF_HEAL.md`
- `01-KINETIC-ENGINE/08_PILLARS_AND_TOOL_IMPLEMENTATION_BOUNDARIES.md`
- `01-KINETIC-ENGINE/09_MCP_RUNTIME_AND_ORCHESTRATOR_REMAINING.md`
- `01-KINETIC-ENGINE/09_PILLARS_STATE_MEMORY_FEEDBACK_AND_SAFETY_AUDIT.md`
- `01-KINETIC-ENGINE/10_TOOLS_POLICY_SCHEMAS_AND_COMMAND_RISK.md`
- `01-KINETIC-ENGINE/10_TOOL_EXECUTORS_INTENT_AND_POLICY_CROSSCHECK.md`
- `01-KINETIC-ENGINE/11_INTENT_PILLARS_AND_REMAINING_TOOLS.md`
- `01-KINETIC-ENGINE/11_ORCHESTRATION_EXECUTION_AUTHORITY_AND_CURRENT_REPO_REALITY.md`
- `01-KINETIC-ENGINE/12_MCP_CONTRACT_AND_ENGINE_CROSS_LAYER_AUDIT.md`
- `01-KINETIC-ENGINE/12_MCP_DISPATCH_AND_EXECUTION_PIPELINE.md`
- `01-KINETIC-ENGINE/13_ENGINE_RUNTIME_BUILD_AND_REGRESSION_AUDIT.md`
- `01-KINETIC-ENGINE/13_ENGINE_SOURCE_TREE_AND_TEST_AUDIT.md`
- `01-KINETIC-ENGINE/14_BUILD_RUNTIME_ARTIFACTS_AND_FINAL_CLOSURE.md`
- `01-KINETIC-ENGINE/14_RUST_TS_STREAM_BOUNDARY_AND_FINAL_CLOSURE_AUDIT.md`
- `01-KINETIC-ENGINE/15_20_TOOL_CATALOG_RUNTIME_MATRIX.md`
- `01-KINETIC-ENGINE/15_TOOLS_IMPLEMENTATION_RECONCILIATION.md`
- `01-KINETIC-ENGINE/16_TOOL_CALLSITE_AND_AGENT_LOOP_RECONCILIATION.md`
- `01-KINETIC-ENGINE/16_TOOL_DISPATCH_AUTHORITY_RECONCILIATION.md`
- `01-KINETIC-ENGINE/17_ENGINE_SURFACE_ROUTING_AND_EXECUTION_AUTHORITY.md`
- `01-KINETIC-ENGINE/17_TOOL_CATALOG_POLICY_DRIFT.md`
- `01-KINETIC-ENGINE/18_ENGINE_FINAL_CLOSURE_AND_INVENTORY.md`
- `01-KINETIC-ENGINE/18_FINAL_SECURITY_AND_END_TO_END_AUDIT.md`
- `01-KINETIC-ENGINE/19_FINAL_PUBLIC_SURFACE_CALLER_AND_SECURITY_RECONCILIATION.md`

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

The durable Engine records document the implementation of MCP, Orchestrator, Indexer/Retrieval, Tools, Intent, Pillars, execution ledger, progress/streaming, Team indexer, runtime/build artifacts, and regression guardrails. They distinguish implemented behavior from stubs, placeholders, legacy paths, and historical build evidence.

### Final Engine reconciliation

The final gate reconciled the public Orchestrator surface, MCP bridge primitives, tool catalog and implementations, major callers, Rust agent-loop dispatch, Core/host RPC call sites, colocated intent/tool tests, cross-module regression guardrails, and the principal security chain. The detailed closure is recorded in `19_FINAL_PUBLIC_SURFACE_CALLER_AND_SECURITY_RECONCILIATION.md`.

## Engine architecture assessment boundary

The Engine is architecturally strong enough to proceed to Core study. Production hardening remains a separate engineering activity. In particular, standalone SDK exposure of low-level filesystem/network helpers should add explicit workspace/SSRF/path-boundary tests, and fresh runtime/build/deployment validation should be performed before making high-assurance production claims.

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

## Current phase pointer
**Phase 2 — Kinetic Core**

Engine is closed for the architecture study. Kinetic Core is now the active phase. Engine assumptions should be reopened only when a Core call site requires specific validation.
