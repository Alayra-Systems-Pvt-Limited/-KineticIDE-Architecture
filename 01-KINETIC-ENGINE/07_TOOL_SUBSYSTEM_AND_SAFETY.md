# Kinetic Engine — Tool Subsystem and Safety Model

## Status
PARTIAL. This pass traced the tool module surface, policy, command-risk classifier, schema catalog, image-generation/vision-asset helpers, and the primary filesystem/process operations. Remaining tool execution functions and all pillar integrations still require cross-reference before the engine is closed.

## Source
`extensions/kinetic-core/kinetic-engine/src/tools/`

## 1. Tool subsystem role

The tool subsystem is the engine's controlled boundary between model-generated actions and host/workspace effects. It contains:

- primitive file and directory operations;
- process/command execution;
- website crawling and screenshot acquisition;
- local project serving;
- debug-session persistence;
- image generation and asset conversion;
- tool catalog/schema generation;
- lane policy;
- command-risk classification;
- MCP permission requests.

The important architectural property is that **tool availability, lane policy, risk classification, and execution are separate concerns**.

## 2. Tool catalog as source of truth

`schemas.rs` loads `tools/toolCatalog.json` at compile time. The catalog is parsed once using `OnceLock`.

Each entry carries:

- name;
- category;
- description;
- lane allowlist;
- Sentry risk tier;
- optional risk classifier;
- executor;
- optional bridge method;
- timeout;
- OpenAI-compatible parameter schema.

The current test suite expects **20 catalog entries**.

The engine builds OpenAI-style function schemas from this catalog rather than maintaining an independent hard-coded schema list.

### Lane resolution

```text
Tool Catalog laneAllowlist
             ∩
Mode Policy denylist
             ↓
Tools exposed to model for lane
```

There is a second plan-phase schema builder which further restricts the catalog to `category == read`.

Tests verify that:

- `ask_response` cannot mutate, execute commands, emit pulses, fetch web pages, or capture screenshots;
- `plan_artifact` cannot write, execute, delete, or capture screenshots;
- `code_exec` excludes vision-only tools;
- `vision_clone` includes browser/screenshot interaction tools.

This means **the model's tool menu is itself capability-filtered before execution**.

## 3. Mode policy

`policy.rs` loads `tools/modePolicy.json` as a compile-time embedded SSOT.

The policy defines:

- lanes;
- tool denylist per lane;
- blocked pulse subtypes per lane;
- forbidden write globs;
- plan-artifact allowed write prefixes;
- plan-artifact-specific forbidden globs;
- classifier capability caps.

Policy is cached process-wide.

Path normalization converts Windows separators to `/` and removes leading `./` before glob matching.

### Mutation defense in depth

`validate_workspace_relative_write()` first checks global forbidden globs. For `plan_artifact`, it additionally requires an approved path prefix and evaluates plan-specific forbidden globs.

Plan artifacts are normalized to:

```text
.kinetic/artifacts/<safe-basename>.md
```

with basename sanitization preventing path separators, traversal, and arbitrary characters.

## 4. Filesystem tool boundary

`Tools::read_file()` is a direct asynchronous read.

`read_file_with_options()` supports a metadata-only mode returning existence, byte size, and modification time without reading content.

`read_multiple_files()` returns a JSON array containing either content or an individual file error, allowing partial success instead of aborting the entire batch.

`write_file()` performs two distinct authorization layers:

```text
write_file request
      ↓
Deterministic safety_gate::check_file_write
      ↓
MCP permission request
      ↓
filesystem write
```

This is significant: the MCP permission is **not** the only safety mechanism.

The tool also creates missing parent directories and reports filesystem failure through the MCP progress channel.

## 5. Command execution boundary

`execute_command()` similarly requires:

```text
command
  ↓
safety_gate::check_command
  ↓
MCP permission request
  ↓
platform shell
  ↓
stdout + stderr result
```

Windows uses `cmd /C`; non-Windows uses `sh -c`.

The returned output combines stdout and stderr and labels stderr explicitly.

The implementation therefore treats command execution as an explicitly mediated capability rather than allowing arbitrary model-generated subprocesses directly.

## 6. Sentry command-risk classifier

`command_risk.rs` defines four levels:

- `LOW` / `AutoApprove`
- `MEDIUM`
- `HIGH`
- `BLOCKED`

The classifier was hardened specifically against shell-wrapper bypasses and substring false positives.

### Shell-wrapper recursion

Commands such as:

```text
bash -c 'rm -rf /'
cmd /c "..."
powershell -Command "..."
```

are detected as wrappers, the embedded command is extracted, and the classifier recursively evaluates the wrapped command.

Binary normalization handles both POSIX and Windows executable paths and removes `.exe`.

### Destructive command detection

The blocked set covers destructive Unix, Windows, and PowerShell operations, including filesystem formatting/deletion and system shutdown operations.

It also detects `mkfs.*` variants.

Git force-push is specially examined when directed at `main` or `master`.

### Chained command handling

The classifier splits shell chains on:

- `&&`
- `||`
- `;`
- `|`
- Windows `&`

Each segment is classified independently and the highest risk wins.

Therefore a harmless command followed by a destructive command cannot inherit the harmless command's risk.

### Known explicit classification

Examples encoded by tests include:

```text
`git log --oneline -10` → LOW
`cargo check` → LOW
`npm run build` → MEDIUM
`npm install express` → HIGH
`git log && rm -rf /` → BLOCKED
```

## 7. Vision/browser tool support

The tool module also exposes website crawling and screenshot functionality.

`crawl_website()`:

- accepts only HTTP(S);
- uses an HTTP client with a 25-second timeout;
- strips scripts/styles/tags;
- collapses whitespace;
- truncates extracted text to 24,000 characters.

`take_screenshot()` has two modes:

1. if `KINETIC_SCREENSHOT_URL_TEMPLATE` is configured, it calls an external screenshot API;
2. otherwise it falls back to extracted website text and explicitly reports that no bundled headless browser exists.

This is therefore **not inherently a local pixel-perfect browser implementation**. Pixel capture depends on an externally configured screenshot service.

## 8. Local project serving

`serve_project()` probes `python3` once per process and falls back to `python`. It then launches Python's built-in HTTP server for a specified port and directory.

The binary choice is cached with `OnceLock`.

Windows additionally sets `CREATE_NO_WINDOW`.

## 9. Debug-session persistence

`update_debug_state()` stores per-command state under:

```text
<workspace>/.kinetic/debug_session.json
```

The identity key is:

```text
cwd + "::" + command
```

The persisted record includes command, cwd, last output, and timestamp.

`read_debug_state()` returns the JSON state or a no-state message.

This is a lightweight local persistence mechanism rather than a database-backed execution history.

## 10. Image generation routing

`image_generation.rs` implements a unified image-generation path using session BYOK credentials.

Endpoint normalization converts a supplied base URL into an OpenAI-compatible `/v1/images/generations` endpoint.

The OpenAI-compatible request sends:

- model;
- prompt;
- one image;
- 1024×1024;
- URL response format.

The returned image URL is fetched and saved to the requested output path.

Routing behavior:

```text
Anthropic → explicit unsupported
Google Generative Language → HF fallback
OpenRouter → OpenAI-compatible attempt → HF fallback
OpenAI/Azure → OpenAI-compatible generation
Other keyed providers → compatible attempt → HF fallback
No usable key → HF fallback
```

HF fallback uses `HUGGINGFACE_API_KEY` and the FLUX.1-schnell inference endpoint.

## 11. Vision asset generation/conversion

`vision_assets.rs` adds:

- HF FLUX image generation;
- strict generated filename validation;
- workspace path containment checking;
- PNG/JPEG/WebP/ICO conversion;
- SVG→PNG rendering with `resvg`/`tiny-skia`;
- SVG→ICO conversion;
- wrapper-SVG generation for raster assets.

Generated filenames must be basenames, are limited to 120 characters, reject traversal/separators, and accept only image extensions.

Asset conversion paths are checked against the workspace root before I/O.

## 12. Architectural security model discovered

The tool layer uses multiple independent controls:

```text
                    Model request
                         │
                 Lane tool exposure
                         │
                  Mode policy
                         │
                 Tool schema/catalog
                         │
              Command/path risk checks
                         │
               Deterministic safety gate
                         │
                 MCP permission gate
                         │
                 Host-side execution
```

Not every tool necessarily traverses every layer identically; this diagram represents the control-plane intent and the confirmed mutation/command paths must be cross-checked in the remaining dispatch code.

## 13. Important implementation distinctions

Confirmed implemented:

- catalog-driven tool schema generation;
- lane filtering;
- policy denylist;
- forbidden path globbing;
- plan-artifact path confinement;
- command risk classification;
- shell-wrapper recursion;
- chained-command max-risk evaluation;
- file reads/writes;
- command execution;
- website text crawling;
- configurable screenshot API integration;
- local Python project serving;
- debug-state persistence;
- OpenAI-compatible image generation;
- HF fallback;
- local image conversion.

Not claimed complete yet:

- all 20 catalog executor implementations;
- complete dispatch mapping from every catalog executor to implementation;
- every pillar's enforcement point;
- all bridge-backed tools;
- full tool timeout behavior;
- complete orchestrator-to-tool approval lifecycle.

Those are required before declaring the tool subsystem complete.
