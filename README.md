# KineticIDE Architecture Study

> The durable architecture and implementation study of the existing **Kinetic IDE** codebase.

## What this repository is

This is a **study and architecture repository**, not the Kinetic IDE source repository.

Its purpose is to preserve implementation-level understanding of Kinetic IDE so the architecture can be studied deeply, resumed after long gaps, and used as a durable reference without modifying the original product repository.

**Source repository:** `Alayra-Systems-Pvt-Limited/KineticIDE`

**Study rule:** the Kinetic IDE source repository is treated as read-only during this study. No source file, configuration, documentation, generated artifact, or other byte is modified, deleted, renamed, or reformatted as part of the study.

The study proceeds layer-by-layer:

1. **Kinetic Engine** — execution, orchestration, tools, MCP, indexing/retrieval, safety, runtime and persistence boundaries.
2. **Kinetic Core** — host-side integration, commands/events/APIs, state and coordination around the Engine.
3. **Kinetic UI** — user-facing workflows and their end-to-end connection to Core and Engine.
4. **Cross-layer audit** — reconstruct important features from UI entry point to Engine behavior and back to the result.

This repository is intended to prevent architectural context from being lost between conversations, agents, or long periods of work.

---

## What is Kinetic IDE?

**Kinetic IDE** is a sovereign, local-first AI development environment built around a **Rust execution engine, vector RAG system, and multi-model orchestration**.

The product is designed around user sovereignty and local-first operation rather than making the developer environment dependent on a remote cloud control plane. The source repository describes support for:

- Local execution with encrypted data storage
- Multiple AI models, including BYOK, local models, and API keys
- Real-time code execution with zero-trust security gates
- Vector-based code understanding through RAG
- Enterprise opt-in forensic auditing

The product repository is proprietary and contains the Workbench foundation together with Kinetic's Engine, Core, UI, AI bridging, MCP, security, and RAG components.

---

# Kinetic Engine

## What it is

The **Kinetic Engine** is the execution and intelligence runtime underneath Kinetic IDE. It is not simply an API wrapper around an AI model. It provides the runtime boundary through which AI-driven development work can be planned, executed, governed, indexed, observed, and returned to the host/UI.

At a high level, the studied architecture is:

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

## Stack and major boundaries

The current study identifies the following major implementation layers and technologies:

- **Rust** — core execution engine and high-performance runtime components.
- **TypeScript host/Core boundary** — integration with the Workbench/extension host and coordination with the Engine.
- **MCP-style bridge** — NDJSON / JSON-RPC-like communication between host-side components and the Rust Engine.
- **Orchestrator** — execution control plane, intent/lane handling, AI client/provider interaction, tool execution, progress and result lifecycle.
- **Indexer/Retrieval subsystem** — workspace indexing and semantic retrieval for code understanding.
- **Vector/RAG infrastructure** — vector retrieval, graph-oriented retrieval, embeddings and BM25-style retrieval paths.
- **LanceDB / Apache Arrow** — storage/data infrastructure identified in the Engine architecture for vector/index data.
- **Tool policy and Sentry controls** — deterministic command-risk classification and host/security permission gates around tool execution.
- **Execution Ledger** — durable execution-oriented accounting/state surface used by the runtime lifecycle.

These descriptions are based on the implementation study and are intentionally separated from future roadmap claims.

---

## Why the Engine has architectural value

The strongest architectural property observed so far is that Kinetic Engine treats AI development as a **controlled runtime problem**, not merely a text-generation problem.

The Engine brings several concerns into one execution architecture:

1. **Model interaction** — provider/client abstraction rather than coupling the IDE to one model.
2. **Agent execution** — orchestration of multi-step work instead of a single request/response cycle.
3. **Tool execution** — explicit tool catalog, policy, command-risk classification and Sentry/host permission controls.
4. **Codebase understanding** — indexing and retrieval are part of the runtime architecture rather than an unrelated external search service.
5. **Execution state** — progress, streaming, results and an execution ledger give the runtime a lifecycle beyond a stateless model call.
6. **Local-first operation** — the architecture is compatible with the product's sovereignty and local execution goals.
7. **MCP boundary** — the host/Engine interface creates a separable runtime boundary between the TypeScript environment and the Rust execution system.

The important point is not that each individual component is novel in isolation. The potential value is in **how these components are composed into one local-first, security-conscious AI development runtime**.

---

## Engine study status

The Engine has been deeply studied and its major architecture is documented in the numbered records under [`01-KINETIC-ENGINE/`](./01-KINETIC-ENGINE/).

The master ledger deliberately keeps the Engine in **final closure** until the completion gates are explicitly verified. In particular, the final gate requires reconciliation of every remaining Rust file/module, colocated tests and guardrails, tool implementations and call sites, public Orchestrator/MCP/tool methods, feature-to-implementation mappings, security boundaries, and the principal end-to-end execution and indexing flows.

This distinction is intentional: **deeply studied does not mean the closure checklist is allowed to be marked complete prematurely.**

### Existing Engine study records

- `01_ENTRYPOINT_AND_RUNTIME.md`
- `02_MCP_BRIDGE_AND_DISPATCH.md`
- `03_INDEXER_SUBSYSTEM.md`
- `04_INDEX_BUILD_RETRIEVAL_STORAGE.md`
- `05_EXECUTION_AI_TOOLS_AND_RUNTIME_SUPPORT.md`
- `06_ORCHESTRATOR_CONTROL_PLANE_PART1.md`
- `07_ORCHESTRATOR_EXECUTION_LIFECYCLE.md`
- `08_ORCHESTRATOR_EXECUTION_AND_SELF_HEAL.md`
- `09_MCP_RUNTIME_AND_ORCHESTRATOR_REMAINING.md`
- `10_TOOLS_POLICY_SCHEMAS_AND_COMMAND_RISK.md`
- `11_INTENT_PILLARS_AND_REMAINING_TOOLS.md`
- `12_MCP_CONTRACT_AND_ENGINE_CROSS_LAYER_AUDIT.md`
- `13_ENGINE_SOURCE_TREE_AND_TEST_AUDIT.md`
- `14_BUILD_RUNTIME_ARTIFACTS_AND_FINAL_CLOSURE.md`

---

# Kinetic Core — reserved study section

**Status: Not yet studied.**

Once the Engine closure gate is complete, this README will be extended with the implementation study of `extensions/kinetic-core`, including its architecture, responsibilities, APIs/events/commands, state management, Engine integration, and end-to-end feature participation.

No Core conclusions are recorded here yet so that this document does not claim knowledge that has not been verified from the source.

---

# Kinetic UI — later study section

The UI will be studied after Core. Its section will document the actual UI architecture, major workflows, state/data boundaries, Core integration, and the end-to-end path from user interaction through Core and Engine.

---

# End-to-end architecture goal

The final architecture record should allow a future engineer to answer, for every significant feature:

```text
User/UI entry
    ↓
Kinetic UI
    ↓
Kinetic Core command / event / API
    ↓
MCP / Engine boundary
    ↓
Engine subsystem
    ↓
Provider / tool / filesystem / index / local dependency
    ↓
State / persistence / execution ledger
    ↓
Result / error / stream
    ↓
Core
    ↓
UI
```

The goal is not merely to document folder names. It is to reconstruct **what exists, where it lives, how it is called, what state it changes, what security controls it passes through, what remains incomplete, and how the pieces behave together in production**.

---

## Architectural assessment principle

The final engineering assessment will distinguish three things:

- **Implemented and verified** — directly supported by the source study.
- **Architecturally strong but requiring production hardening** — a sound design whose operational maturity still needs validation.
- **Potential / future capability** — a plausible extension that is not currently implemented.

This prevents the architecture study from confusing an ambitious design with a feature that already exists in production.

---

## Source of truth

The implementation source remains:

`Alayra-Systems-Pvt-Limited/KineticIDE`

This repository is the durable **architecture-study ledger** for that source. It does not replace the source code and must not be treated as a fork of the product.

**Study order:** Engine → Core → UI → cross-layer audit.
