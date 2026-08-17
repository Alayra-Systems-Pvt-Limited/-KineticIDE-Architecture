# Kinetic Engine — Tool Executors, Intent Routing, and Policy Cross-Check

## Status
**PARTIAL — major executor and intent behavior traced. Final engine closure still requires the complete orchestrator call graph, MCP dispatch, remaining tests/configuration, and repository-wide cross-check.**

## 1. Catalog is the current tool contract

`extensions/kinetic-core/tools/toolCatalog.json` is the practical registry/SSOT referenced by the engine schema builder and the human-readable mode matrix.

Current catalog contains 20 tools across read, write, exec, pulse, vision, and assets categories.

The matrix explicitly says Rust `schemas.rs` and TypeScript `toolRouter.ts` derive from this catalog. Lane changes are treated as breaking changes and are validated with `npm run verify:tool-matrix`.

Current lane contract:

| Lane | Capability |
|---|---|
| `ask_response` | read/search/analyze only |
| `plan_artifact` | read + web fetch + planning pulse restrictions |
| `code_exec` | full execution except Vision-only browser tools |
| `vision_clone` | code execution + vision tools |

The source matrix currently says `emit_pulse` is only catalog-allowlisted for `code_exec` and `vision_clone`; Plan's historical footnote describes a runtime restriction rather than an allowlist entry in the current catalog. This is a useful source-of-truth discrepancy to retain and verify against current TypeScript routing during the Core study.

## 2. Rust executor implementation

`tools/mod.rs` is a concrete executor collection, not merely an interface.

### Read operations

- `read_file`
- `read_file_with_options`
- `read_multiple_files`
- `list_directory_json`
- `list_dir`
- `search_files`
- `grep_search`

`search_files` uses Rust regex + `ignore::WalkBuilder`, respects `.gitignore`, skips `.git`, `node_modules`, and `target`, skips files over 1 MiB, and caps matches at 200.

`grep_search` similarly uses pure Rust regex and `ignore`, but accepts an explicit maximum result count. The presence of both APIs indicates historical/compatibility overlap that should be resolved through the catalog/dispatch call graph rather than assumed to be two distinct product capabilities.

### File mutation

`replace_in_file` requires exactly one occurrence and retries writes up to five times with increasing short delays.

`create_directory` uses recursive directory creation.

`move_file` first attempts native rename and falls back to recursive copy/delete when rename fails. Directory fallback is implemented recursively.

`delete_path` and `delete_file` both invoke the deterministic pillar safety gate before filesystem deletion.

The duplicate delete APIs are another call-graph item for final audit.

### Command execution

`execute_command`:

```text
command
  ↓
Pillar safety_gate::check_command
  ↓
MCP permission request
  ↓
platform shell
  ↓
stdout/stderr aggregation
```

Windows uses `cmd /C`; Unix-like systems use `sh -c`.

Important: the direct Rust executor shown here does not itself impose the catalog's declared 120-second timeout; timeout enforcement must therefore be verified at the orchestrator/dispatch layer.

`read_result` provides a structured success/failure summary for a completed process and intentionally truncates output to the first 20 lines in its summary.

## 3. Vision executor reality

`capture_screenshot` calls `take_screenshot`.

Without `KINETIC_SCREENSHOT_URL_TEMPLATE`, `take_screenshot` does **not** produce a real screenshot: it returns a text extraction fallback and explicitly states that no headless browser is bundled.

With the environment template configured, it requests bytes from an external screenshot service and returns a byte-count summary. `capture_screenshot` can write that summary to the requested path, not necessarily the raw screenshot bytes.

`snapshot_ui` is simply website text extraction.

`ui_interact` is HTTP-level behavior:

- click/type/scroll → GET page and return content preview + requested intent;
- submit → POST supplied value.

It does not operate a browser DOM and does not target the VS Code UI.

Therefore the current `vision_clone` capability is **vision-oriented but not a full local browser automation runtime**.

## 4. Asset executors

`generate_image_unified` routes by provider base URL:

```text
Anthropic → unsupported
Google Generative Language → HF fallback
OpenRouter → OpenAI-compatible attempt → HF fallback
OpenAI/Azure → OpenAI-compatible generation
Other keyed endpoints → compatible attempt → HF fallback
No usable key → HF fallback
```

OpenAI-compatible generation posts to `/v1/images/generations`, requests one 1024×1024 image, obtains a returned URL, downloads the bytes, and writes them to the requested path.

HF fallback uses `HUGGINGFACE_API_KEY` and FLUX.1-schnell.

`vision_assets.rs` additionally provides workspace path containment, filename sanitization, PNG/JPEG/WebP/ICO/SVG conversion, SVG rasterization, and image-to-SVG wrapping.

## 5. Intent classifier

`intent/classifier.rs` is the engine's intent-to-lane decision logic.

### Intent taxonomy

```text
Informational
Explore
Artifact
CodeChange
VisionClone
```

### Execution lanes

```text
AskResponse
PlanArtifact
CodeExec
VisionClone
```

### Classification signals

The classifier combines three weighted signals:

```text
Structural       65%
Workspace        15%
Momentum         20%
```

Structural analysis includes:

- question/opening patterns;
- vision markers;
- imperative code verbs;
- planning markers;
- exploration markers;
- full-body code-verb scan;
- casual/greeting handling.

Workspace context can push toward code when diagnostics exist or an active code file is open.

Momentum can push toward CodeChange when recent intents are predominantly code work.

### Attachment override

Attachments normally force `VisionClone`, except analysis/review/test-style language causes a safety fallback to `Informational`.

This is an explicit anti-misrouting mechanism.

### User-mode capability ceiling

`resolve_lane()` applies the user's selected mode as a hard ceiling:

```text
ask    → AskResponse
plan   → AskResponse / PlanArtifact
code   → AskResponse / PlanArtifact / CodeExec
vision → AskResponse / PlanArtifact / CodeExec / VisionClone
```

For example, a VisionClone intent in Code mode is capped to `CodeExec` and records `cap_applied=true`.

This is stronger than simply selecting tools after classification: the **execution lane itself is clamped before tool exposure**.

## 6. Ambiguity handling

The classifier has deliberate mode-specific adjustments.

In Code mode, a low-confidence Informational result below `0.60` can be upgraded to CodeChange, unless the attachment-analysis failsafe applies.

In Plan mode, explicit requests to save/write/export Markdown artifacts can convert Informational/Explore to Artifact.

In Code/Vision mode, an Artifact result caused by incidental plan/blueprint language can be converted back to CodeChange when implementation verbs are also present.

These rules indicate the classifier is designed around **dashboard mode intent**, not merely generic NLP classification.

## 7. Mission builder

`intent/mission_builder.rs` is deliberately simpler than the classifier and explicitly states that intent classification is now handled centrally by `classifier.rs`.

It converts a prompt into executable mission hints:

- CreateFile
- ModifyFile
- RunCommand
- InstallPackage
- General

It detects package-install keywords, file-extension mentions, and otherwise creates a general mission from the first six words.

The implementation is heuristic and should not be confused with the primary lane classifier.

## 8. Safety and routing relationship

The discovered architecture is:

```text
User prompt
    ↓
Intent classifier
    ↓
TaskIntent
    ↓
User-mode capability ceiling
    ↓
ExecutionLane
    ↓
Catalog laneAllowlist / mode policy
    ↓
Tool schema exposure
    ↓
Tool executor
    ↓
Deterministic safety gate (mutation/command paths)
    ↓
MCP permission (where applicable)
    ↓
Host effect
```

This establishes several independent gates rather than trusting model-selected tool names.

## 9. Remaining questions to close before Engine completion

1. Exact orchestrator dispatch path from model tool-call name to every one of the 20 catalog executors.
2. Where catalog timeout values are enforced, especially `execute_command` (120s) and image generation (90s).
3. Whether duplicate Rust APIs (`list_dir` vs `list_directory_json`, `grep_search` vs `search_files`, `delete_file` vs `delete_path`) are legacy, compatibility, or active dispatch targets.
4. Exact TypeScript bridge behavior for the catalog's TypeScript executors.
5. Exact `emit_pulse` runtime restrictions and whether current docs/catalog are synchronized.
6. Complete MCP request/response and cancellation path for tool execution.
7. Remaining engine tests, Cargo configuration, and feature flags.

These are intentionally left as open audit items rather than guessed.
