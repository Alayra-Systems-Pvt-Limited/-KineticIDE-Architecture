# Kinetic Engine — Intent, Pillars, and Remaining Tool Surface

## Study status
This document records the current source-level study of the remaining small tool modules plus the Intent and Pillars modules. The overall engine remains **PARTIAL** until the full file inventory, orchestrator/MCP/tool integration, tests, and cross-module audit are reconciled.

## 1. Remaining tool modules

The current `src/tools/` directory contains these files:

- `browser.rs`
- `command_risk.rs`
- `image_generation.rs`
- `mod.rs`
- `policy.rs`
- `pulse_tools.rs`
- `sandbox.rs`
- `schemas.rs`
- `vision_assets.rs`

`browser.rs`, `pulse_tools.rs`, and `sandbox.rs` are explicit stubs/reserved modules rather than active implementations.

### Image generation
`image_generation.rs` implements a unified session-BYOK image-generation route.

The request carries:

- provider base URL;
- API key;
- model ID;
- prompt;
- output path;
- timeout.

OpenAI-compatible image generation normalizes the endpoint to `/v1/images/generations`, submits `model/prompt/n/size/response_format`, downloads the returned URL, and writes bytes to the requested output path.

Routing rules include:

- Anthropic → explicit unsupported error;
- Google Generative Language host → Hugging Face fallback;
- OpenRouter → try OpenAI-compatible generation, then HF fallback;
- OpenAI/Azure → OpenAI-compatible generation when key exists;
- other keyed hosts → try compatible generation, then HF fallback;
- missing key → HF fallback.

This establishes a provider-agnostic image capability with a concrete HF FLUX fallback rather than an image provider being hard-coded at the agent layer.

### Vision assets
`vision_assets.rs` provides two major functions.

1. **HF image generation** using `HUGGINGFACE_API_KEY` and FLUX.1-schnell.
2. **Workspace-safe asset conversion**.

Filename sanitization rejects empty names, names over 120 characters, path separators, `..`, invalid characters, and unsupported extensions. Supported raster output extensions are PNG/JPG/JPEG/WebP.

Workspace path sanitization is performed before asset conversion and checks both the initial path components and canonicalized result against the workspace root. This is an explicit traversal/escape defense.

Supported conversion operations include:

- PNG/JPEG → WebP
- resize
- SVG → PNG using `usvg` + `resvg` + `tiny-skia`
- PNG → ICO
- SVG → ICO
- catalog-oriented conversion to PNG/JPEG/WebP/ICO/SVG

The SVG target conversion for raster input creates a self-contained base64 image wrapper SVG.

## 2. Intent architecture

`intent/mod.rs` exposes:

- `classifier`
- `mission_builder`

The classifier is the **single routing authority** for user intent → execution lane.

### TaskIntent

Current intents:

- `Informational`
- `Explore`
- `Artifact`
- `CodeChange`
- `VisionClone`

### ExecutionLane

Current lanes:

- `AskResponse`
- `PlanArtifact`
- `CodeExec`
- `VisionClone`

The critical design is the capability ceiling:

```text
UserMode × TaskIntent → ExecutionLane
```

`resolve_lane()` caps the requested intent according to the selected user mode:

- `ask` caps everything to AskResponse;
- `plan` allows information/exploration and artifacts, but caps code/vision to PlanArtifact;
- `code` allows CodeExec but caps VisionClone to CodeExec;
- `vision` allows VisionClone;
- unknown/default behaves like code mode.

This lane decision is consumed by the orchestrator's execution controls, so the classifier is not merely UI labeling.

### Classification signals
`classify_with_context()` combines three sources:

1. structural prompt signal;
2. workspace signal;
3. session momentum.

The structural classifier recognizes questions, vision markers, interrogative openers, imperative code verbs, planning/artifact language, exploration language, body-level code verbs, and casual/greeting language.

Workspace signals include diagnostics and active-code-file extensions. Session momentum examines recent intents and can reinforce CodeChange.

Attachments have a failsafe override: analysis/review/audit language with an attachment stays informational, while a generic attachment becomes VisionClone.

There are additional routing exceptions for Plan-mode Markdown artifact requests and Code-mode prompts containing incidental planning words.

### Mission builder
`mission_builder.rs` converts a prompt into a simple serializable mission list.

Mission types:

- CreateFile
- ModifyFile
- RunCommand
- InstallPackage
- General

It extracts package-install commands and file-extension mentions. If nothing specific is found it creates one General mission from the first six prompt words.

Important boundary: the mission builder is a **lightweight mission decomposition helper**, not the main intent authority. The source explicitly states that classification now lives in `classifier.rs`.

## 3. Pillar architecture

`pillars/mod.rs` exposes six modules:

- `state_tracker`
- `safety_gate`
- `context_manager`
- `memory_bridge`
- `feedback_signal`
- `risk_engine`

### Pillar 02 — State tracker

`state_tracker.rs` persists session execution state to:

```text
<workspace>/.kinetic/debug_session.json
```

The state records:

- session ID;
- workspace;
- created files;
- modified files;
- commands run;
- commands passed/failed;
- current and total steps;
- last update timestamp.

It provides read/write helpers plus mutation helpers and clear-state.

This is simple filesystem JSON state, distinct from the execution ledger/trace systems documented elsewhere.

### Pillar 03 — Session context

`context_manager.rs` defines an in-memory `SessionContext` containing:

- files in scope;
- recently written files;
- recently read files;
- active file.

`add_written()` is active. `add_read()`, `set_active()`, and `clear()` are explicitly annotated as reserved/unwired scaffolding in the source.

It also implements a heuristic semantic-neighbor system without LSP/Tree-Sitter:

- same-directory source siblings;
- naming relationships such as Service/Controller/Model/Provider/Handler;
- bounded neighbor count.

`build_neighbor_context()` reads up to 30 lines from up to five neighbors and respects a character budget before producing a prompt-ready context block.

Therefore this pillar currently mixes a partly-wired session context model with an active lightweight neighbor heuristic.

### Pillar 04 — Memory bridge

`memory_bridge.rs` persists cross-session facts to:

```text
<workspace>/.kinetic/memory.json
```

Trace files live under:

```text
<workspace>/.kinetic/pulses/
```

Facts carry source, timestamp, and session ID. Duplicate facts are suppressed and the store is bounded to the latest 50 facts.

`build_memory_context()` injects recent facts in a `<memory>` block.

Startup trace distillation scans the newest three `TRACE_*.md` files and extracts lines beginning with:

- Created
- Modified
- Fixed
- Installed
- Built

Those extracted lines are persisted as memory facts.

This is deterministic text extraction, not an LLM memory process.

### Pillar 05 — Feedback signal

`feedback_signal.rs` persists feedback to:

```text
<workspace>/.kinetic/feedback.json
```

Feedback types are:

- Correction
- Preference
- Rejection
- Approval

Currently explicit recording helpers are provided for corrections and preferences. Corrections are bounded to the latest 100 entries. Preferences are deduplicated by correction text.

`build_feedback_context()` prioritizes preferences and recent corrections and formats them into a `<feedback>` prompt block.

This creates a durable local feedback channel but does not itself constitute model fine-tuning.

### Pillar 07 — Safety gate

`safety_gate.rs` is a deterministic hard gate.

It contains blocked destructive command patterns and protected path patterns. File deletion is blocked by default except for a small explicit cache/temp allowlist:

- `.kinetic/temp/`
- `node_modules/.cache/`
- `.next/cache/`

The command and write checks are independent from higher-level model instructions. The delete check is fail-closed unless the path matches the explicit safe list.

This is one layer of the engine's broader safety architecture; `tools::command_risk` and tool policy provide additional controls.

### Pillar 06 — Risk engine

`risk_engine.rs` is currently explicitly marked `allow(dead_code)` and states that Phase 20 uses `tools::command_risk` instead.

It defines a future risk model:

- LOW — safe development commands;
- MEDIUM — standard workspace writes;
- HIGH — destructive or external access.

It checks external path access, destructive command signatures, known safe command signatures, workspace artifact operations, and falls back to MEDIUM for uncertain commands.

This module is therefore **future/reserved infrastructure**, not the currently authoritative command-risk implementation.

## 4. Important architectural distinction

There are multiple safety/risk layers and they should not be collapsed into one concept:

```text
Intent classifier
      ↓
Execution lane / mode cap
      ↓
Tool policy
      ↓
Tool-specific risk classification
      ↓
Safety gate / host permission boundaries
      ↓
OS operation
```

Separately, state, memory, feedback, and session context feed information back toward orchestration/prompt construction.

## 5. Remaining engine work after this pass

Still required before declaring the engine complete:

1. Finish remaining `orchestrator.rs` branches and reconcile every public execution path.
2. Complete `mcp.rs` method-by-method contract study.
3. Fully trace `tools/mod.rs`, `command_risk.rs`, `policy.rs`, and `schemas.rs` call sites rather than only their primary definitions.
4. Verify every tool catalog/schema contract against the TypeScript-side registry.
5. Inspect all engine tests and fixtures.
6. Inspect engine build/runtime configuration and scripts.
7. Reconcile all root `src/` files against the study ledger.
8. Produce the final engine dependency graph and end-to-end feature matrix.
9. Explicitly classify implemented vs scaffolded/dead vs transitional code.
