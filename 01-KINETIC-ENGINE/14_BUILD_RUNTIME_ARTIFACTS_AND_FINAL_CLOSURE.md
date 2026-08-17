# Kinetic Engine — Build, Runtime Artifacts, Tool Closure, and Final Closure Pass

## Scope
This pass closes the remaining explicitly enumerated tool files and audits the engine package-level runtime/build artifacts. Source repository remains read-only for this study.

## 1. Complete `tools/` file inventory

The current `extensions/kinetic-core/kinetic-engine/src/tools/` directory contains these ten files:

1. `browser.rs`
2. `command_risk.rs`
3. `image_generation.rs`
4. `mod.rs`
5. `policy.rs`
6. `pulse_tools.rs`
7. `sandbox.rs`
8. `schemas.rs`
9. `vision_assets.rs`

The directory listing currently exposes nine files in the returned API payload; the module source is the authoritative exported surface. The study therefore treats `mod.rs` plus the explicitly enumerated files as the current tool module set and records the three tiny placeholder modules separately.

### Explicit placeholders

- `browser.rs`: stub for future Playwright-driven browser automation.
- `pulse_tools.rs`: stub for future Pulse emission helpers; explicitly not wired into the tool registry.
- `sandbox.rs`: stub for future execution sandbox / VM SDK integration.

These files exist but do not constitute active capabilities.

## 2. Image-generation path

`image_generation.rs` implements session-BYOK image generation with Hugging Face fallback.

The primary path is OpenAI-compatible `POST /v1/images/generations`, normalized from the session `base_url`. The request sends model, prompt, one image, 1024×1024 size, and URL response format, then downloads the returned URL and writes bytes to the requested output path.

Provider routing:

- Anthropic → explicit unsupported error.
- Google Generative Language host → HF fallback.
- OpenRouter → try OpenAI-compatible generation, then HF fallback.
- OpenAI/Azure → OpenAI-compatible generation when an API key is present.
- Other keyed endpoints → try OpenAI-compatible generation, then HF fallback.
- Missing key → HF fallback.

HF fallback requires `HUGGINGFACE_API_KEY` and delegates to `vision_assets::generate_image_hf`.

This is a real capability, not a stub. It is provider-aware but intentionally uses a generic OpenAI-compatible image contract where possible.

## 3. Vision asset pipeline

`vision_assets.rs` provides two distinct capabilities.

### Generation
Uses Hugging Face serverless FLUX.1-schnell and writes returned image bytes to a workspace path.

### Conversion
`convert_asset_sync` supports:

- PNG/JPEG → WebP
- resize
- SVG → PNG via `usvg` + `resvg` + `tiny-skia`
- PNG → ICO
- SVG → ICO

`convert_asset_for_catalog` adds a higher-level catalog contract for target formats `png`, `jpeg`, `webp`, `ico`, and `svg`.

### Workspace safety
Generated/conversion filenames are sanitized and restricted to basename-like values for generated names. Conversion source/destination paths are checked against the workspace root before and after canonicalization, preventing path traversal/escape through relative or absolute paths.

## 4. Tool safety conclusion

The active tool architecture remains layered:

```text
model-visible schema
      ↓
mode/lane policy
      ↓
tool dispatcher
      ↓
deterministic command risk / path validation
      ↓
host Sentry / permission boundary where applicable
      ↓
OS operation
```

The existence of `browser.rs` and `sandbox.rs` does not imply those capabilities are available. They are placeholders and must remain classified as such.

## 5. Package/build architecture

`Cargo.toml` identifies the engine as the `kinetic-engine` Rust 2021 sidecar. Major runtime dependencies include Tokio, Serde, Reqwest, LanceDB, Arrow, FastEmbed, Git2, Tree-sitter language grammars, Petgraph, Axum/Tower, CPAL/Hound, image/resvg/usvg/tiny-skia, and system telemetry.

Important build properties:

- default build is CPU-only;
- optional `cuda` feature enables FastEmbed CUDA support;
- release profile optimizes for size (`opt-level=z`), enables LTO, one codegen unit, stripping, and `panic=abort`;
- Windows has `windows-sys` dependency;
- Unix has `libc` dependency.

This confirms the engine is designed as a cross-platform sidecar with optional GPU acceleration rather than a browser-only TypeScript service.

## 6. Release/build script

`build_release.bat` is Windows-oriented. It force-kills `kinetic-engine.exe`, waits one second, runs `cargo build --release`, and tells the developer to reload VS Code.

This is a developer-local replacement/deployment workflow, not a production updater.

## 7. Team indexer production container

`docker/Dockerfile.indexer` builds the Rust engine from only its Cargo manifests and `src/`, then copies the release binary into a Debian Bookworm slim runtime image.

Runtime image details:

- installs `ca-certificates`, `libssl3`, `libgit2-1.5`;
- creates unprivileged user `kinetic` UID 10001;
- exposes port 9100;
- defaults `KINETIC_SERVE_BIND=0.0.0.0:9100`;
- starts `kinetic-engine serve`.

This is specifically the Team indexer deployment, not the complete desktop IDE.

## 8. Team indexer compose stack

`docker-compose.indexer.yml` mounts the team repository read-only at `/repo`, persists Lance data at `/var/kinetic`, requires repo ID and API key environment variables, polls Git every 300 seconds, and exposes port 9100.

The design therefore separates:

- immutable source repository mount;
- mutable vector/index storage volume;
- secret configuration;
- long-lived indexer process.

## 9. Historical build/compile artifacts

The engine directory also contains:

- `build_error.log`
- `build_error.txt`
- `compile_errors.txt`
- `err.log`
- `result.log`
- `Cargo.lock`
- `protoc.zip`
- `protoc/`

The text logs are historical development artifacts and must **not** be interpreted as the current build state without checking their commit/source correspondence.

The inspected historical logs demonstrate prior dependency/API and source integration failures, including:

- a LanceDB `FullTextSearchQuery` visibility/API mismatch;
- an older `main.rs` missing `workspace_path` binding;
- an older source state missing the `uuid` dependency;
- ordinary unused-variable/dead-code warnings.

The current `Cargo.toml` does contain `uuid`, so the historical `uuid` error is not evidence of the current source being in the same state.

The binary `protoc.zip`/`protoc` artifacts are build-support assets rather than Rust source and are recorded as binary artifacts; their internal binary contents are not treated as source architecture.

## 10. Test-tree finding

No separate `extensions/kinetic-core/kinetic-engine/tests/` directory is currently exposed by the repository API. The engine instead contains substantial inline `#[cfg(test)]` coverage and `regression_guardrails.rs` architecture-regression tests in `src/`.

Therefore the correct model is:

```text
engine tests
  ├── inline module tests
  └── regression_guardrails.rs
```

not a separate integration-test tree.

## 11. Current closure assessment

The engine study now has source-level coverage of:

- entrypoint/runtime;
- MCP bridge and dispatch;
- orchestration control plane and execution lifecycle;
- intent classification/mission building;
- tool policy/schema/risk and remaining tool implementations;
- pillars;
- workspace indexing;
- hybrid retrieval;
- vector storage;
- graph retrieval;
- hashing/name indexing/symbol extraction;
- AI client/model capabilities;
- code execution/ledger;
- progress/GPU/runtime support;
- Team indexer HTTP/runtime;
- build/package configuration;
- Docker deployment;
- historical build artifacts;
- regression guardrails.

### Still required before final `ENGINE COMPLETE`

A final cross-reference audit must compare every exported public symbol and major subsystem entrypoint against its callers in `kinetic-engine`, `kinetic-core`, and the host boundary. This is specifically to prevent a seemingly complete source-file study from missing a stale/dead compatibility path.

The final master architecture should only be marked complete after that audit.
