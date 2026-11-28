---
phase: 01-operation-first-document-model
plan: 04
type: summary
wave: 4
---

# 01-04 Summary — Wave 4: hostile input, fixture migration, 0.2.0 release

## Task 1 — Hostile-input tests and fixture migration

- `tests/it/hostile.rs` (new, wired via `mod hostile;` in `tests/it/main.rs`): malformed JSON
  -> `DocError::Parse`; non-stencilcraft header -> `DocError::NotAStencilcraftFile`; v3 header ->
  `DocError::FormatVersion`; unknown top-level fields ignored; duplicate shape ids ->
  `DocError::Model`; dangling member id pruned on load; `power = INFINITY` / `passes = 0` fail
  `save_json` with `DocError::Model`. All assertions use `assert!(matches!(...))`; no
  unwrap/expect/panic on hostile input.
- `tests/it/migrate.rs` extended: `poster_migrates_to_three_shapes_with_cut_and_engrave_layers`
  (3 shapes, Cut[1,2] + Engrave[3], next_shape_id 4) and
  `stencil_frame_migrates_to_two_shapes_and_preserves_kerf` (2 shapes, one Cut layer [1,2],
  next_shape_id 3, kerf 0.2). tool-passes assertions (4 shapes, 4 layers, next_shape_id 5,
  distinct Score passes and unmerged Engrave passes) were already present from 01-02.

## Task 2 — Release metadata

- `Cargo.toml`: `[workspace.package] version` 0.1.0 -> 0.2.0; the `=0.1.0` path pins on
  `craft-pdf`/`craft-ps` updated to `=0.2.0` (workspace crates, required for resolution).
- `Cargo.lock`: regenerated via `cargo metadata`; only workspace crate versions changed
  0.1.0 -> 0.2.0, no registry upgrades.
- `CHANGELOG.md`: `## v0.2.0 — 2026-11-24` with v1 before / v2 after, legacy read-only compat
  fields (D-16), in-memory v1 migration with no v2 downgrade (D-13).
- `stencilcraft-format/README.md`: describes the v2 envelope, `FORMAT_VERSION` 2,
  `SUPPORTED_VERSIONS` [1, 2].
- `stencilcraft-document/README.md`: describes the geometry-once `Document.shapes` model.

## Verification

- All grep marker checks pass (version, CHANGELOG heading, both Cargo.lock entries, both README literals).
- `cargo test -p craft-stencilcraft-format -p craft-stencilcraft-document --locked`: 26 + 30 + 11 passed, 0 failed.
- `cargo test -p craft-stencilcraft-document --test it --locked`: 30 passed, 0 failed.
- `cargo clippy -p craft-stencilcraft-format -p craft-stencilcraft-document --all-targets --locked -- -D warnings`: clean.
