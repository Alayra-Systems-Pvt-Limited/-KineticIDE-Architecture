# Kinetic Engine — Tools: Policy, Schemas, and Command Risk

## Status
**PARTIAL.** This pass establishes the tool-control architecture from `tools/mod.rs`, `policy.rs`, `schemas.rs`, and `command_risk.rs`. Remaining tool implementations and the complete catalog still require inspection.

## 1. Tool subsystem architecture

`tools/mod.rs` is the central native tool facade. It exposes read/write filesystem operations, web crawling/screenshot support, local HTTP serving, command execution, debug state, directory listing, and search. It also re-exports policy, schemas, image generation, and vision asset helpers.

Important cross-cutting controls are applied at the native execution boundary:

```text
LLM tool call
   ↓
Tool schema / lane exposure
   ↓
Policy + lane checks
   ↓
Native executor
   ├── Pillar safety gate (where applicable)
   ├── Sentry / permission request
   └── filesystem/process/network operation
```

The engine therefore does not rely solely on the model being given a safe tool list; mutating native operations have additional runtime controls.

## 2. Filesystem tools

`read_file` reads asynchronously. `read_file_with_options` supports metadata-only responses containing existence, byte size, and modification time, avoiding content transfer when metadata is sufficient.

`read_multiple_files` processes requested files sequentially and returns an array containing either content or a per-file error, so one missing file does not abort the entire batch.

`write_file` performs a deterministic Pillar 07 safety check before writing. It resolves the target relative to the current process directory, creates missing parent directories, requests host permission through the MCP bridge, and only then writes. Filesystem errors are surfaced through MCP progress and returned as failures.

This is a significant distinction: **write permission is not simply an LLM decision; it is mediated by safety gate + host permission + actual filesystem operation.**

## 3. Web/vision support

`crawl_website` only accepts HTTP(S), uses a bounded 25-second request timeout, strips scripts/styles/tags, normalizes whitespace, and caps extracted text at 24,000 characters.

`take_screenshot` supports an externally configured screenshot API through `KINETIC_SCREENSHOT_URL_TEMPLATE`. Without that configuration there is no bundled headless browser; it falls back to extracted page text and explicitly tells the caller that a manual screenshot or configured screenshot service is required for pixel-perfect cloning.

This means the vision tool contract is intentionally dependency-light rather than silently claiming local browser screenshot capability.

## 4. Local serving and command execution

`serve_project` discovers `python3` vs `python` once per process using `OnceLock`, then starts Python's built-in HTTP server. Windows receives the `CREATE_NO_WINDOW` process flag.

`execute_command` applies the Pillar 07 command safety gate, requests host permission through MCP, then launches:

- Windows: `cmd /C`;
- non-Windows: `sh -c`.

Stdout and stderr are combined into a single returned result. This is the execution primitive used by the higher-level orchestration/self-healing path documented separately.

## 5. Debug state

`update_debug_state` persists per-command state under:

```text
<workspace>/.kinetic/debug_session.json
```

The identity key is `cwd::command`. Stored state includes command, cwd, last output, and timestamp. `read_debug_state` retrieves the persisted JSON or returns a no-state message.

This gives the execution/self-healing system durable local evidence from prior command attempts.

## 6. Directory listing and search

`list_directory_json` canonicalizes the target where possible, enumerates metadata, and sorts entries by name. Returned records include name, directory flag, size, and modification time.

`search_files` builds a regex and optional glob filter, establishing a native discovery primitive used by planning and retrieval-adjacent flows. The complete implementation and its limits still need to be traced from the remaining `mod.rs` sections.

## 7. Mode policy is a single source of truth

`policy.rs` embeds `tools/modePolicy.json` at compile time. A process-global `OnceLock` caches the parsed policy.

The policy contains:

- policy version;
- lane definitions;
- tool denylist by lane;
- blocked Pulse subtypes by lane;
- forbidden write globs;
- Plan artifact write prefixes;
- Plan-specific forbidden write globs;
- classifier capability rules.

The important design is **catalog allowlist ∩ policy denylist**. A tool being present in the catalog does not automatically mean it is exposed in every lane.

## 8. Path safety policy

Workspace-relative paths are normalized by converting backslashes to `/` and stripping leading `./` segments.

Global forbidden-write globs are compiled into a `GlobSet` and applied to mutating paths.

The `plan_artifact` lane has an additional restriction: writes must fall under configured approved prefixes, with extra hot-source/manifest deny globs applied even inside otherwise permitted artifact roots.

Plan markdown filenames are sanitized to a basename containing only ASCII alphanumeric characters plus `.`, `-`, `_`, with `.md` enforced. The canonical artifact location is:

```text
.kinetic/artifacts/<safe-name>.md
```

This prevents traversal through suggested filenames and keeps Plan artifacts in a controlled workspace subtree.

## 9. Lane-specific tool schemas

`schemas.rs` embeds `tools/toolCatalog.json` as compile-time SSOT and caches the parsed 20-entry catalog.

Each entry contains:

- name/category/description;
- lane allowlist;
- Sentry tier/classifier;
- executor;
- optional bridge method;
- optional timeout;
- OpenAI-style parameter schema.

`build_tool_schemas_for_lane()` returns OpenAI Chat Completions function schemas after applying catalog lane allowlist and policy denylist.

`build_plan_phase_tool_schemas_for_lane()` further restricts the plan phase to catalog entries categorized as `read`.

The tests encode the intended lane isolation:

- `ask_response` excludes workspace mutation, execution, web fetch, and screenshot tools;
- `plan_artifact` excludes mutation, execution, deletion, and screenshot tools;
- `code_exec` excludes vision-only tools;
- `vision_clone` includes screenshot/UI interaction tools.

This is a strong separation between **tool declaration** and **tool execution**: the same catalog is used to construct the model-facing interface while native policy remains authoritative at runtime.

## 10. Command risk classifier

`command_risk.rs` defines four risk levels:

```text
AutoApprove < Medium < High < Blocked
```

The classifier was hardened against several attack classes.

### Shell-wrapper recursion

It recognizes shell wrappers including:

- bash
- sh
- zsh
- dash
- ash
- cmd
- powershell
- pwsh

Binary names are normalized to strip path prefixes and `.exe`. Wrapper forms such as `bash -c`, `cmd /c`, and PowerShell command forms are unwrapped and recursively classified.

This prevents a destructive command hidden inside a shell wrapper from receiving the wrapper binary's otherwise benign classification.

### Tokenized destructive detection

The blocked check operates on command tokens rather than naive substring matching. This prevents text such as `cargo run -- "rm -rf"` from being classified merely because a destructive string appears inside an argument.

Destructive classes include Unix tools (`rm`, `shred`, `dd`, `mkfs` and `mkfs.*`), Windows destructive commands (`del`, `format`, `diskpart`, `attrib`, destructive `rmdir`/`rd` forms), PowerShell destructive operations, and shutdown/restart operations.

Git force-pushes to `main`/`master` are also treated as blocked when the relevant force flags are present.

### Chain classification

The classifier recognizes shell chaining via `&&`, `||`, `;`, `|`, and Windows single `&`. Each segment is classified independently and the maximum risk wins. Therefore a benign command followed by a blocked command remains blocked.

### Default tiers

Known read/build commands have explicit lower classifications:

- `git log/status/diff/...`, `cargo check/clippy/tree`, selected npm/test/lint commands, version probes, `pwd`, `ls`, `cat`, etc. → `AutoApprove`.
- `cargo build`, `npm run build/compile/watch`, `tsc`, `webpack`, `vite build` → `Medium`.
- Unrecognized commands → `High`.
- Destructive matches → `Blocked`.

The classifier is therefore intentionally conservative for unknown commands.

## 11. Cross-layer invariant

The current tool safety architecture is layered:

```text
Tool catalog
   ↓
Lane allowlist
   ↓
Mode policy denylist
   ↓
Orchestrator execution lane
   ↓
Native tool
   ↓
Pillar deterministic safety gate
   ↓
MCP host permission / Sentry
   ↓
OS operation
```

The precise combination differs by tool, so later study must verify every mutating executor individually rather than treating this diagram as universal.

## 12. Remaining tool study

Still required before declaring `tools/` complete:

- remaining `mod.rs` functions after the fetched section;
- complete `image_generation.rs`;
- complete `vision_assets.rs`;
- all remaining catalog entries and `toolCatalog.json`;
- `pulse_tools.rs`;
- `browser.rs`;
- `sandbox.rs`;
- integration of tools with orchestrator and intent lanes;
- every tool's actual executor/bridge path;
- tests and error semantics.
