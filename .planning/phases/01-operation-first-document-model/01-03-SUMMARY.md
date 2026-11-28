---
phase: 01-operation-first-document-model
plan: 03
wave: 3
status: complete
---

# 01-03 Summary — Derived queries, layer deletion, pruning, validation, tests

## What was implemented

**`src/lib.rs`** — new `Document` methods and hardened validation:
- `unassigned_shapes()` — shapes in no operation layer; Guide membership counts as assigned (D-04).
- `production_sequence()` — layers with `produces()` in document order, each with its member shapes in membership order (D-03/D-17/D-19).
- `layer_bounds(layer_index)` — union of member shape bounds.
- `remove_layer(layer_index)` — removes layer + memberships, never touches `shapes`; last-layer removal allowed (MODEL-07, OPEN QUESTION 7).
- `prune_members()` — drops dangling/duplicate member ids, first-seen order preserved; called in `load_json_named` before `validate`, and in `save_json` on a validated clone.
- `validate()` split into `validate_model()` + `validate_canvas()`; model now also rejects `next_shape_id` not above all shape ids and member refs to missing shapes, and names the offending layer.

**`src/production.rs`** — `Operation::validate` additionally rejects negative power/speed; all numerics finite-checked, `passes >= 1`, negative kerf/tab width rejected, all errors mapped to `DocError::Model`.

**Tests** — `tests/it/{derived,operations,canvas}.rs` added, registered in `main.rs`.

## Verification

- `cargo test -p craft-stencilcraft-document --locked`: 26 unit + 21 integration passed, 0 failed.
- `cargo test ... --test it --locked`: 21 passed.
- `cargo clippy ... --all-targets --locked -- -D warnings`: clean.
- `cargo fmt ... -- --check`: clean.
