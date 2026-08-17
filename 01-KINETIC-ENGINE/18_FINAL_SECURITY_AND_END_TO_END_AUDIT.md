# Kinetic Engine — Final Security and End-to-End Audit

## Status
**FINAL ENGINE CLOSURE — AUDIT PASS.**

This document consolidates the security/process boundaries and primary end-to-end flows established by the source study. It does not override the remaining module-by-module completion gate in the Engine README.

## 1. Agent execution flow

```text
TypeScript Core / UI
      ↓ processInstruction
MCP stdin/stdout NDJSON
      ↓
McpServer
      ↓
Orchestrator::execute_plan
      ↓
Intent classifier
      ↓
ExecutionLane capability ceiling
      ↓
context/session construction
      ↓
provider request / streaming
      ↓
structured tool-call recovery
      ↓
policy + tool schema validation
      ↓
legacy structured loop OR Rust-owned agent loop
      ↓
Tools / host bridge
      ↓
state tracker + execution ledger
      ↓
final summary constrained by completed ledger
      ↓
MCP stream/result
      ↓
Core/UI
```

## 2. Command security

For Rust-owned agent-loop commands:

```text
command
 ↓
classify_command_risk
 ├─ Blocked
 ├─ AutoApprove / LOW
 ├─ Medium
 └─ High
 ↓
Sentry review when session is not trusted
 ↓
MCP host execute_command
 ↓
state_tracker
```

The command-risk classifier is therefore an active deterministic authority for the new agent loop.

The legacy structured command path has a separate autonomous command execution/self-heal implementation and host permission path. Both must remain documented as distinct compatibility paths.

## 3. Filesystem security

Protected mutation layers include:

- lane/path policy;
- forbidden write globs;
- workspace-relative validation for agent writes;
- `pillars::safety_gate` for native file mutation primitives;
- Sentry/host approval for high-impact legacy mutations;
- TypeScript `propose_edits` gate for the agent-loop write path.

Sensitive path policy explicitly covers:

- `.env`
- `.env.*`
- `id_rsa`
- `id_ecdsa`
- `.ssh/**`

Plan artifacts have separate permitted/forbidden prefixes/extensions.

## 4. Network security

Engine network surfaces include provider requests, website crawling, screenshot API access, Hugging Face image generation, and Team indexer HTTP.

Observed controls include:

- HTTP(S)-scheme checks for website tools;
- request timeouts;
- bounded page/text responses;
- Team bearer authentication;
- Team API-key prefix validation;
- provider credential passed in request headers;
- explicit provider-specific image-generation routing.

The full SSRF policy for the TypeScript `web_fetch` executor is a Core-side boundary and must be audited during Core study; the engine delegates that tool to TypeScript.

## 5. Process boundary

The engine is a sidecar process. MCP owns the IPC contract, including request correlation, pending-response lifecycle, bounded output streaming, interrupt behavior, and timeout cleanup.

This means the host remains a privileged execution boundary for operations that should not be performed blindly inside Rust.

## 6. Indexing flow

```text
Core / MCP index_workspace
        ↓
single-flight job key
        ↓
WorkspaceIndexer
        ↓
File traversal / hashing / symbol extraction
        ↓
embedding
        ↓
LanceDB / Arrow tables
        ↓
symbol graph / metadata
        ↓
retrieval ready
```

Updates/deletes converge on the same indexing implementation and invalidate retrieval caches.

## 7. Retrieval flow

```text
Core / MCP query
      ↓
indexer::retrieve
      ↓
revision/cache key
      ├─ cache hit → SearchOutcome
      └─ miss → QueryEngine
                   ├─ vector search
                   ├─ BM25
                   ├─ RRF
                   └─ GraphRAG expansion
      ↓
path filtering / Top-K
      ↓
RetrieveContextResult
      ↓
MCP / Orchestrator context
```

Indexed Git revision is compared with current HEAD and surfaced as stale metadata rather than silently treating the index as current.

## 8. Team indexer flow

```text
Team HTTP client
      ↓ bearer authentication
      ↓ repo ID validation
      ↓
health / status / query / trigger
      ↓
shared VectorEngine
      ↓
WorkspaceIndexer / retrieve
```

Boot indexing and Git polling use the same single-flight build path. Shard registration remains a deliberate `501` placeholder.

## 9. Execution ledger trust boundary

The model can propose a plan, but it cannot define the authoritative history of what happened.

```text
planned tool calls
      ≠
completed tool actions

completed_ledger
      ↓
final summary clamp
```

This is a significant anti-hallucination architecture feature.

## 10. Security findings requiring Core verification

The Engine study identifies several controls whose final correctness depends on Core:

1. Sentry review implementation and default-deny behavior.
2. TypeScript `propose_edits` write gate.
3. TypeScript `web_fetch` SSRF enforcement.
4. Tool-router catalog enforcement.
5. Session trust persistence/clear behavior.
6. Core-side permission cancellation and timeout semantics.

These are intentionally carried into the Core study rather than falsely marked as Engine-only concerns.

## 11. Final audit conclusions

Confirmed architecture properties:

- model tool choice is not the sole authorization mechanism;
- command risk is deterministic;
- lane policy limits capabilities before execution;
- high-impact operations can cross a host approval boundary;
- file mutations use additional local policy/safety controls;
- index writes converge on a canonical indexer;
- retrieval is revision-aware and cache-aware;
- agent summaries are constrained by a completed-action ledger;
- legacy and new agent execution paths coexist and must be distinguished;
- several declared features are explicit stubs rather than silently assumed implemented.

## Remaining completion gate

The only acceptable next step before Engine can be marked COMPLETE is a final source-ledger reconciliation confirming that every file/module and relevant colocated test has been read and that the feature/security/call-graph records cover each one. Any file that cannot be verified must be marked `BLOCKED`, not inferred.
