# Kinetic Engine — 20-Tool Catalog Runtime Matrix

## Status
**CLOSURE AUDIT — catalog and current TypeScript router directly cross-checked.**

The current catalog contains exactly 20 entries. The catalog is the schema/lane SSOT consumed by Rust schema generation and TypeScript routing.

| # | Tool | Catalog executor | Current Rust/TS route observed | Closure status |
|---|---|---|---|---|
| 1 | `read_file` | Rust | explicit Rust/TS route | Verified |
| 2 | `read_multiple_files` | Rust | Rust implementation exists; final dispatch ownership still checked | Partial |
| 3 | `list_directory` | Rust | Rust implementation exists; naming/dispatch still checked | Partial |
| 4 | `search_files` | Rust | Rust implementation exists; final dispatch ownership still checked | Partial |
| 5 | `get_editor_state` | TypeScript | `handleGetEditorState` | Verified |
| 6 | `get_diagnostics` | TypeScript | `handleGetDiagnostics` | Verified |
| 7 | `write_file` | Rust | explicit route + JIT gate | Verified |
| 8 | `replace_in_file` | Rust | Rust implementation exists; final dispatch ownership still checked | Partial |
| 9 | `create_directory` | Rust | Rust implementation exists; final dispatch ownership still checked | Partial |
| 10 | `move_file` | Rust | Rust implementation exists; final dispatch ownership still checked | Partial |
| 11 | `delete_file` | Rust | explicit route + policy + JIT gate | Verified |
| 12 | `execute_command` | Rust | explicit route; Rust safety gate | Verified |
| 13 | `web_fetch` | TypeScript | explicit route + Sentry + `handleWebFetch` | Verified |
| 14 | `emit_pulse` | TypeScript | explicit route; lane-gated | Verified |
| 15 | `capture_screenshot` | Rust | Rust executor exists; current TS router does not directly switch-dispatch it | Needs Rust dispatch cross-check |
| 16 | `snapshot_ui` | Rust | Rust executor exists; current TS router does not directly switch-dispatch it | Needs Rust dispatch cross-check |
| 17 | `ui_interact` | Rust | Rust executor exists; current TS router does not directly switch-dispatch it | Needs Rust dispatch cross-check |
| 18 | `generate_image` | Rust | Rust executor exists; current TS router does not directly switch-dispatch it | Needs Rust dispatch cross-check |
| 19 | `convert_asset` | Rust | Rust executor exists; current TS router does not directly switch-dispatch it | Needs Rust dispatch cross-check |
| 20 | `propose_edits` | TypeScript | explicit route + canonical callback | Verified |

## Important distinction

`toolRouter.ts` is **not the complete 20-tool executor registry**. It is the TypeScript-side compatibility/UI router. Rust-owned tools may be executed directly by the engine and returned over JSON-RPC.

Therefore absence from the TypeScript switch is not evidence that a Rust tool is unimplemented. It must be reconciled with the Rust orchestrator/tool-dispatch path.

## Confirmed lane model

- Read tools are available across the four lanes according to their individual allowlists.
- Mutation and command tools are restricted to `code_exec` and `vision_clone`.
- `web_fetch` begins at `plan_artifact`.
- Vision tools are `vision_clone` only.
- Asset generation/conversion are `code_exec` + `vision_clone`.
- Editor state/diagnostics are TypeScript bridge capabilities.
- `propose_edits` feeds the canonical VS Code diff/Sentry path.

## Sentry model

Catalog tiers are not uniformly equivalent to automatic approval. `execute_command` uses the dynamic command-risk classifier; mutation/network tools can additionally use JIT Sentry paths where the bridge owns the UI approval lifecycle.

## Closure conclusion

The 20-tool catalog itself is verified. Remaining work is **proof of dispatch ownership** for Rust-owned entries and exact timeout enforcement at the actual dispatch boundary. This prevents incorrectly treating the TypeScript router as the sole tool implementation registry.
