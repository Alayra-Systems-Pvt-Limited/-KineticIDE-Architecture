# Kinetic Engine — Tool Dispatch Authority Reconciliation

## Status
Important closure finding. The catalog is complete and lane-aware, but the source shows a deliberate split between Rust executor ownership and the TypeScript compatibility/UI router. The study must distinguish catalog declaration, implementation, and reachable runtime dispatch.

## Catalog
`toolCatalog.json` contains exactly 20 entries. Rust `schemas.rs` parses the same file and tests `len() == 20`. TypeScript `toolRegistry.ts` also treats it as the 20-tool SSOT. Each entry declares `executor` as Rust or TypeScript, with optional bridge method and timeout.

## TypeScript router reality
`ToolRouter.route()` explicitly handles:
- `write_file`
- `read_file`
- `execute_command`
- `read_result`
- `emit_pulse`
- `propose_edits`
- `delete_file`
- `get_editor_state`
- `get_diagnostics`
- `web_fetch`

`read_result` is an internal compatibility/result protocol tool and is not one of the 20 catalog entries. Therefore `toolRouter.ts` is not the complete 20-tool executor registry.

## Rust executor reality
`tools/mod.rs` implements the Rust-owned catalog surface including `read_file`, `read_multiple_files`, directory/list/search operations, `write_file`, `replace_in_file`, `create_directory`, `move_file`, delete operations, command execution, screenshot/page/UI helpers, image generation, and asset conversion.

A Rust function existing is not by itself proof of model reachability; its orchestrator dispatch path must be verified.

## Timeout closure item
The catalog declares:
- `execute_command` = 120s
- `web_fetch` = 15s
- `generate_image` = 90s

Rust `schemas.rs` exposes `timeout_secs_for_tool()`, but the current source search has not yet established a broad dispatch call site consuming this helper. Implementation-level network timeouts are separate and do not prove catalog timeout enforcement.

## Lane/security architecture
Rust builds model-visible tool schemas from the catalog and policy. TypeScript independently builds lane maps and gates calls at the bridge boundary. This is deliberate defense in depth:

```text
Rust intent/lane
   ↓
Rust tool exposure
   ↓
NDJSON / JSON-RPC
   ↓
TS lane gate
   ↓
TS Sentry/UI effects
```

Mutation paths can have both Rust deterministic safety checks and TypeScript mode-policy/JIT Sentry checks.

## Final Engine closure task
The remaining work is now a reachability audit for each of the 20 catalog entries:

```text
catalog declaration
 → executor owner
 → Rust dispatch OR TS bridge dispatch
 → lane gate
 → safety/approval gate
 → timeout
 → result path
```

This is the final mechanical audit before Engine can be marked COMPLETE. No implementation changes are being made to KineticIDE during this architecture study.
