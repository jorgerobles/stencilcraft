# 01-01 Summary — v2 envelope + operation-first model + v1 migration (Wave 1)

## Implemented

**craft-stencilcraft-format** (`crates/stencilcraft-format/src/lib.rs`)
- `FORMAT_VERSION = 2`, new `SUPPORTED_VERSIONS = &[1, 2]`.
- `DocumentFields` gained a 12th type param `Shapes` with a `shapes` field (`default` + `skip_serializing_if`); `"shapes"` added to `is_wire_field`.
- New `read_versioned`/`read_versioned_from` and `read_typed_versioned`/`read_typed_versioned_from` returning `(version, File)`; shared `header_version` helper; strict `read`/`read_typed` still accept only v2 (old builds reject v2, D-13).

**craft-stencilcraft-document**
- `production.rs`: `Operation` is now a 10-variant internally tagged (`kind`, kebab-case) data enum (Cut/KissCut/DieCut/Draw/Score/Engrave/Perforate/PrintThenCut/Punch/Guide) with `OperationSettings` (power, speed, passes, focus, kerf, tabs, tip, deboss, punch diameter), `Tab`, `Process`, `Strategy`, and `process()`/`strategy()`/`is_guide()`/`validate()` (failures are `DocError::Model`). `ShapeProduction.operation` and `Shape::operation()` removed; `ToolOp`/`Tip` kept as legacy read-only.
- `wire_v1.rs` (new): deserialize-only v1 mirrors (`V1Layer`, `V1Shape`, `V1ShapeProduction`, `V1SourceGroup`, `V1Operation`).
- `migrate.rs` (new): `OperationDefaults` (D-15 neutral seed) + crate-visible `migrate_v1_layers` grouping by full `PassKey` (`ToolOp` included, D-14), template layers → `Operation::Guide` (D-17), geometry deduplicated once, intra-layer order preserved. Inline tests prove two Score tips and Emboss vs PowerEngrave stay in separate layers.
- `lib.rs`: `Layer` = name/operation/preset_id/preset_linked/visible/locked/color/members (no `shapes`/`kind`/`template`/`bounds`); `Shape` lost `style`/`tool`; `Document.shapes` added; default document is one operation-less `base` layer (D-18); `push_shape` stores geometry once + records membership; `load_json_named` dispatches on the versioned header (v1 migrates in memory, v2 parses), `load_json_with_seed` added; validate rejects duplicate shape ids and invalid operation settings; dangling members pruned on load.
- `canvas.rs`: `shape_locked`/`layer_of_shape`/`pruned_groups`/`page_document`/`render_thumbnail` reworked to membership lookups against `Document.shapes`.

## Verification

- `cargo test -p craft-stencilcraft-format -p craft-stencilcraft-document --locked`: **37 passed, 0 failed** (format 11, document 26).
- `cargo clippy … --all-targets --locked -- -D warnings`: 0 errors.
- `cargo fmt --check`: clean.
- v1 fixtures load via migration and re-save with header version 2 and a top-level `shapes` array.
