# Kinetic Engine — Tool Catalog / Policy / Documentation Drift

## Verified discrepancy
Three artifacts describe the `emit_pulse` Plan-lane policy differently:

### `extensions/kinetic-core/tools/toolCatalog.json`
`emit_pulse` has:

- laneAllowlist: `code_exec`, `vision_clone`
- executor: TypeScript
- no Plan allowlist

### `extensions/kinetic-core/tools/modePolicy.json`
Explicitly denies `emit_pulse` in `plan_artifact`:

```json
"tool_lane_denylist": {
  "plan_artifact": ["emit_pulse"]
}
```

It also states the Plan lane has no execution pulses.

### Generated policy matrix
`extensions/kinetic-core/docs/KINETIC_MODE_POLICY_MATRIX.md` agrees with the catalog + denylist result: `emit_pulse` is absent from Plan.

### Human-readable `docs/KINETIC_MODE_TOOL_MATRIX.md`
This older/human-readable matrix lists `emit_pulse` as `Yes` for `plan_artifact` and then explains a runtime subtype block. That is inconsistent with the catalog and generated policy matrix.

## Architectural conclusion
The effective implementation contract appears to be:

```text
catalog allowlist
      ∩
modePolicy denylist
      ↓
effective lane tool set
```

Therefore Plan does **not** have `emit_pulse` in the current effective tool set, and the human-readable matrix is stale relative to the generated policy artifacts.

This is documentation/configuration drift, not a reason to infer a different runtime behavior. The validator/generator and actual engine/bridge behavior should be treated as the implementation path to verify during the Core study.

## Why this matters to the architecture study
This is exactly why the study cannot mark Engine complete merely from reading Rust modules. Tool behavior is a cross-layer contract involving:

- `toolCatalog.json`
- `modePolicy.json`
- Rust `schemas.rs`
- Rust `policy.rs`
- Rust `orchestrator.rs`
- TypeScript `toolRouter.ts`
- generated policy documentation
- human-readable product documentation

The final architecture ledger must preserve this distinction and flag drift rather than silently selecting one source.
