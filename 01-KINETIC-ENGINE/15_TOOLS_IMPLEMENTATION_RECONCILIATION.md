# Kinetic Engine — Tools Implementation Reconciliation

## Status
**FINAL CLOSURE IN PROGRESS.** This pass verifies the concrete `tools/mod.rs` implementation and the small nested tool modules. It corrects an earlier assumption that the tool surface was already fully reconciled.

## Concrete tool implementation

`tools/mod.rs` is the main native tool implementation rather than a thin registry. It directly owns filesystem, search, command, web/page, serving, debug-state, and result-formatting operations.

### Filesystem

Implemented primitives include:

- `read_file`
- `read_file_with_options`
- `read_multiple_files`
- `write_file`
- `list_directory_json`
- `list_dir`
- `replace_in_file`
- `create_directory`
- `move_file`
- `delete_path`
- `delete_file`

`write_file`, `delete_path`, and `delete_file` invoke the deterministic `pillars::safety_gate` before mutation. `write_file` additionally requests host/Sentry permission through MCP before performing the write.

`replace_in_file` intentionally requires exactly one match before replacing. It retries the write operation up to five times after the first failure, with increasing short delays.

`move_file` prefers filesystem rename and falls back to recursive copy/remove when rename is unavailable, including directory moves.

### Search

Two native search surfaces exist:

- `search_files(pattern, search_root, file_pattern)` — structured JSON results, regex + optional glob filter, Git-aware traversal, skips `.git`, `node_modules`, `target`, ignores files above 1 MiB, and caps matches at 200.
- `grep_search(pattern, search_path, max_results)` — human-readable grep-style output using the `ignore` crate, with a caller-supplied result cap and a 1 MiB file-size guard.

Neither relies on shell `grep`.

### Command execution

`execute_command` performs the local deterministic safety-gate check and then asks the host through MCP for permission. It uses `cmd /C` on Windows and `sh -c` on Unix-like systems.

The command-risk classifier is a separate layer. It is not automatically equivalent to every direct call of `Tools::execute_command`; orchestration/schema paths must therefore be reconciled separately to determine which model-originated calls pass through `classify_command_risk` before reaching the primitive.

### Debug state

`update_debug_state` and `read_debug_state` persist command/cwd/output/timestamp information under `<workspace>/.kinetic/debug_session.json`.

The identity key is `cwd::command`, so repeated execution of the same command in the same working directory updates one record.

### Web/page operations

Implemented behavior includes:

- `crawl_website` — HTTP(S) GET, script/style/tag stripping, whitespace normalization, 24,000-character output cap.
- `take_screenshot` — uses `KINETIC_SCREENSHOT_URL_TEMPLATE` when configured; otherwise explicitly falls back to page text because no headless browser is bundled.
- `capture_screenshot` — stores the screenshot summary returned by `take_screenshot` when an output path is supplied.
- `snapshot_ui` — currently delegates to `crawl_website`.
- `ui_interact` — currently performs HTTP GET context retrieval for click/type/scroll actions and HTTP POST for `submit`; it does not control VS Code UI and explicitly reports this limitation.

This is important for the feature map: the source contains UI/browser-named tool APIs, but they are not equivalent to full browser automation.

### Local project serving

`serve_project` probes once per process for `python3`, otherwise uses `python`, then launches `python -m http.server` against the requested directory. Windows uses `CREATE_NO_WINDOW`.

### Tool result normalization

`ToolResult` and `read_result` convert process output into a bounded human-readable success/failure summary. Successful output is limited to the first 20 lines; failures expose exit code and a bounded stderr/stdout-derived explanation.

## Image/vision implementation

### `image_generation.rs`

`generate_image_unified` routes by provider base URL:

- Anthropic → explicit unsupported error.
- Google Generative Language endpoint → Hugging Face fallback.
- OpenRouter → OpenAI-compatible image call, then HF fallback.
- OpenAI/Azure → OpenAI-compatible generation when a key exists.
- Other keyed endpoints → attempted OpenAI-compatible generation, then HF fallback.
- No usable key → HF fallback.

The OpenAI-compatible request posts `/v1/images/generations` as needed, requests a URL response, downloads the resulting bytes, and writes them to the requested path.

### `vision_assets.rs`

Provides:

- strict generated filename sanitization;
- Hugging Face FLUX.1-schnell generation;
- workspace-contained path validation;
- PNG/JPEG/WebP conversion;
- resize;
- SVG→PNG rendering through `resvg`/`usvg`/`tiny-skia`;
- PNG/SVG→ICO conversion;
- catalog-oriented format conversion;
- image-to-SVG wrapper generation.

Workspace path validation checks both the lexical component relationship and a second post-canonicalization relationship before returning the path.

## Explicit stubs

Three nested tool files are literal placeholders:

- `tools/browser.rs` — **STUB**, future Playwright-driven browser automation.
- `tools/sandbox.rs` — **STUB**, future execution sandbox/VM SDK integration.
- `tools/pulse_tools.rs` — **STUB**, future Pulse emission helpers and not wired into the tool registry.

These are not missing documentation; they are genuinely unimplemented source files.

## Security boundary discovered

The concrete filesystem/command implementation shows that safety is distributed:

```text
model/tool schema
      ↓
execution lane + tool policy
      ↓
command-risk / plan-level deterministic checks
      ↓
Tools primitive
      ↓
local safety_gate for protected file/command mutations
      ↓
MCP Sentry/host permission where applicable
      ↓
OS operation
```

The layers are complementary, not interchangeable. A future refactor must not collapse them without preserving their separate responsibilities.

## Important implementation gaps / mismatches

1. Browser automation is not implemented despite browser-oriented API names existing elsewhere.
2. Sandbox execution is not implemented.
3. Pulse tool emission is not implemented/wired.
4. Screenshot capture is text fallback unless an external screenshot API template is configured.
5. `ui_interact` is HTTP-level simulation/context retrieval rather than real browser interaction.
6. Direct primitive command execution and command-risk classification are separate concerns; caller tracing is required to prove that every model-originated command reaches the intended risk layer.
7. Several mutation primitives are safety-gated, but the exact gate coverage must still be reconciled across every call site.

## Closure implication

The tool implementation layer is now substantially verified, but Engine completion still requires the final call-site audit: enumerate every caller of each mutating/search/network tool, determine whether the call originates from model-controlled execution, and verify the intended policy/risk/Sentry path for each.
