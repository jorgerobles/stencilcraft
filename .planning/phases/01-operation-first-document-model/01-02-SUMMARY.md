---
phase: 01-operation-first-document-model
plan: 02
wave: 2
status: complete
---

# 01-02 Summary: integration tests for v2 round trip and v1 tool-pass migration

## What was built

- `refs/craft-libs/crates/stencilcraft-document/tests/it/main.rs` — the single `it` test binary (`mod round_trip; mod migrate;`), per rule 30.
- `tests/it/round_trip.rs` — builds a v2 `Document` with a Cut layer (2 shapes), an Engrave layer sharing shape 1 (N:M), and a Guide layer (shape 2); asserts `save_json` writes header version 2, `load_json` returns equal `shapes`/`layers`, the shared geometry is stored once, and the Guide layer survives.
- `tests/it/migrate.rs` — loads all three existing v1 fixtures via `include_str!`, asserts each re-saves as v2 with no `style` key on shapes and no `kind` key on layers; asserts starter yields 3 shapes, Cut[1,2] + Engrave[3], `next_shape_id` 4; asserts the new tool-passes fixture yields 4 layers (2 Score with distinct tip/passes, 2 Engrave — Emboss vs PowerEngrave not merged), each shape in exactly one layer, `next_shape_id` 5.
- `tests/fixtures/tool-passes.stencilcraft` — hand-written v1 fixture: one engrave layer, 4 shapes with tool values score/fine, score/wide-3, emboss, power-engrave, `next-shape-id` 5.

No production source was touched; no new dependencies.

## Verification

- `cargo test -p craft-stencilcraft-document --test it --locked` — 4 passed, 0 failed.
- `cargo clippy -p craft-stencilcraft-document --all-targets --locked -- -D warnings` — clean.
- `cargo fmt -p craft-stencilcraft-document -- --check` — clean.
