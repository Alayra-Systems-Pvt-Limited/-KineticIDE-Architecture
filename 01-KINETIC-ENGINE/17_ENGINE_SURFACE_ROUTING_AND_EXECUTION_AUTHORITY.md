# Kinetic Engine — Surface Routing and Execution Authority

## Status
**CLOSURE PASS — execution-surface routing reconciled.**

## 1. Two different routing decisions

The repository distinguishes execution surface from execution lane.

- **Execution surface:** hosted backend chat replay vs local kinetic-engine.
- **Execution lane:** AskResponse / PlanArtifact / CodeExec / VisionClone inside the engine path.

The routing contract defines:

```text
Ask/Plan + Auto signed in → backend_chat
Code/Vision + Auto signed in → engine
Auto signed out → engine
Custom AI/BYOK → engine
```

When `engine` is selected, the path is `processInstruction → kinetic-engine`.

## 2. Backend chat is not an alternate engine executor

The hosted `backend_chat` branch is remote completion plus `streamBackendResponse` and explicitly has no Rust tool loop for that turn. Tool-execution analysis therefore applies only after the surface router selects `engine`.

## 3. Engine tool authority

The routing contract explicitly protects the canonical `propose_edits` path:

```text
canonical tool type
      ↓
dispatchCanonicalToolCall
      ↓
MCPBridge
      ↓
ToolRouter / applyProposeEditsImpl
      ↓
Sentry + disk + diff UI
```

Legacy parsed-type branches must not become independent implementations. The repository documents that returning `success` without applying edits causes false-positive tool results and empty workspaces.

This confirms that some engine tool results cross back into TypeScript because the host/editor is the authority for applying VS Code edits.

## 4. Final authority model

```text
User turn
   ↓
Surface routing
   ├── backend_chat ──→ hosted response only
   │
   └── engine
         ↓
      intent classifier
         ↓
      execution lane
         ↓
      Rust structured execution
         ├── filesystem/process/indexing effects
         └── host-mediated tools
                ↓
             TypeScript bridge
                ↓
             VS Code/Sentry/diff UI
```

## 5. Closure implication

The engine cannot be understood by looking only at Rust. Its execution authority is intentionally split at the host boundary:

- Rust owns AI orchestration, intent/lane, indexing/retrieval, native filesystem/process tools, execution state, and structured execution decisions.
- TypeScript owns VS Code-specific effects, Sentry UI approval, diff/editor integration, and compatibility routing.
- Hosted backend chat is a separate surface and does not imply local tool execution.

A tool can therefore be cataloged as Rust-owned while still requiring a TypeScript host callback for an effect that only VS Code can perform.

## 6. Remaining Engine closure checks

The execution model is sufficiently reconciled. Remaining work is mechanical repository completeness:

- account for every engine source module/file;
- verify tests/build workflows;
- record genuinely unreachable or legacy code;
- produce the final Engine index/closure document.
