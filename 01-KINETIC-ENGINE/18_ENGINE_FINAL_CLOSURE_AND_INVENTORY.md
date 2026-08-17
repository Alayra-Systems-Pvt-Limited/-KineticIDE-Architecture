# Kinetic Engine — Final Closure and Inventory

## Status
**ENGINE STUDY COMPLETE — architectural understanding is now sufficient to proceed to `kinetic-core`.**

## 1. Engine boundary

The engine is the native Rust execution subsystem under `extensions/kinetic-core/kinetic-engine/`. Its process boundary is the MCP/NDJSON sidecar launched by the extension. TypeScript owns the host/editor lifecycle and communicates with the sidecar.

## 2. Responsibilities

- AI/provider request execution, streaming, turn correlation, bounded orchestration, cancellation, parsing, mission/trace generation, provider health/timeouts.
- Intent classification, confidence, workspace/momentum signals, execution lanes, user-mode capability ceilings, catalog/schema exposure, destructive-operation policy, command-risk classification.
- Filesystem/search/mutation, process execution, vision/assets, image generation/conversion, result normalization and host permission integration.
- Workspace indexing, LanceDB, FastEmbed, BM25/vector retrieval, RRF, GraphRAG/symbol graph, incremental updates, hashing/revisions, query cache and job coalescing.
- Execution ledger, progress/cancellation, session/debug state, memory, trace distillation and feedback persistence.
- MCP sidecar, optional Team indexer server, GPU/embedder startup, release configuration and regression guardrails.

## 3. End-to-end authority

```text
User turn
    ↓
Execution-surface routing
    ├── backend_chat → hosted response only
    └── engine
          ↓ intent classifier
          ↓ mode capability ceiling
          ↓ execution lane
          ↓ catalog/policy
          ↓ Rust orchestration
          ↓ native tool / index / provider effect
             ├── native Rust effect
             └── host-mediated effect → TypeScript → VS Code/Sentry/diff UI
```

Rust is the engine execution authority, but not every physical side effect belongs to Rust. VS Code-specific effects intentionally return to the TypeScript host.

## 4. Execution lanes

`AskResponse`, `PlanArtifact`, `CodeExec`, and `VisionClone` are separate from the surface-routing decision. The selected user mode establishes a hard capability ceiling; model requests cannot elevate that ceiling.

## 5. Safety architecture

Defense in depth consists of surface routing, intent classification, lane ceiling, catalog allowlist, deterministic Rust safety gate, command-risk classification, TypeScript lane gate, JIT Sentry approval where required, and turn/stream stale-data suppression.

## 6. Tool architecture

The catalog contains 20 tools with deliberately mixed ownership. Rust owns native filesystem/process/vision/asset execution; TypeScript owns editor state, diagnostics, web fetch, pulse, and proposal application. The TypeScript `ToolRouter` is not the universal registry.

`propose_edits` is the canonical cross-boundary case: Rust requests/coordinates the operation while TypeScript remains authoritative for Sentry, VS Code edit application, disk mutation and diff UI.

## 7. Vision maturity

Vision exists but is not yet a complete browser-agent runtime. Screenshot support can use an external screenshot endpoint; fallback can be textual extraction; `snapshot_ui` and `ui_interact` are HTTP/content-oriented; image generation/conversion are separate implemented utilities.

## 8. Indexing maturity

The canonical workspace vector pipeline is Lance-based and owns embedding/vector persistence. `FileEnumerator` is a separate filesystem enumeration utility, not a second vector indexer. Regression guardrails protect RRF `k=60`, IVF-PQ threshold, singleton embedder, concurrent BM25/vector retrieval, bounded progress, file-size limits and schema-migration ordering.

## 9. Runtime/build maturity

The engine has a size-optimized release profile, optional GPU acceleration with CPU fallback, configurable global model caching, and reduced OS scheduling priority to avoid starving the IDE host.

## 10. Compatibility surfaces

Deliberate compatibility mechanisms include legacy TypeScript sentinel dispatch, canonical Rust structured execution, normalization of older tool payloads, duplicate/legacy-looking helpers, and pillar modules that are future/legacy infrastructure rather than active authorities.

## 11. Explicit non-claims

This study does not claim every helper is active production dispatch, every catalog capability is a complete standalone product feature, vision is full browser automation, backend chat executes the local tool loop, TypeScript is merely passive transport, or historical roadmap documents necessarily describe current behavior.

## 12. Handoff to Core

The next layer is `kinetic-core`: activation/lifecycle → MCP bridge → routing/tool router → provider/model selection → workspace/indexer bridge → Sentry/security → state/config/persistence → commands/services.

During Core study, engine assumptions should be reopened only when a Core call site requires validation. The engine architecture is now preserved in this folder.

## 13. Completion decision

**COMPLETE FOR ARCHITECTURAL STUDY.**

Remaining engineering questions are refinement, compatibility, or Core/UI concerns rather than blockers to understanding the current engine end-to-end.
