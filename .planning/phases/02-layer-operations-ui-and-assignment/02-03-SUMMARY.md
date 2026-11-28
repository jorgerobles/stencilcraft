---
phase: 02-layer-operations-ui-and-assignment
plan: 03
subsystem: ui
tags: [rust, egui, craft-ui-list-tree, layers-panel, unassigned-group, pass-badges, ui-05, ui-06, ui-07]

# Dependency graph
requires:
  - phase: 02-layer-operations-ui-and-assignment/02-02
    provides: green workspace compile on the v2 model, membership/operation commands, preset catalogue, paint loop already on layer.members
provides:
  - `crates/ui-egui/src/layers.rs` pure tested helpers: `build_layer_rows`, `pass_number`, `effective_operation`, `membership_labels`
  - `shape_visual` takes the owning `&Layer`; `layer.color` (parsed `#rrggbb`) overrides the operation token, unparseable falls back
  - `layers_panel` rebuilt as a `craft_ui::list::List` tree with eye/lock toggles, Pass N badges, membership badges, colour swatch, derived Unassigned group
  - Inspector "Operation" section showing effective operation and source (UI-07)
affects: [02-04 layers panel interactions, 02-05 assignment UI, UI-05, UI-06, UI-07]

# Actuals (#2632)
actuals:
  tokens: 83790
  tasks: 3
  commits: 1

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "All panel display values are pure functions of &Document (layers.rs); the UI pass snapshots rows and dispatches commands only after show()"
    - "Unassigned group and Pass N are derived per frame via Document::unassigned_shapes()/production_sequence(); never persisted (D-09/D-12)"
    - "Layer colour overrides the operation paint token via stencilcraft_doc::canvas::parse_hex_color; kerf preview still wins"

key-files:
  created:
    - worktree crates/ui-egui/src/layers.rs (pure helpers + unit tests)
  modified:
    - worktree crates/ui-egui/src/lib.rs (`pub mod layers`, shape_visual takes &Layer, paint passes layer, 4 new tests)
    - worktree apps/stencilcraft/src/main.rs (tree layers_panel on craft_ui List, PanelKey/RowVisual, open_layer_groups + layer_selection fields, inspector Operation section)
    - worktree apps/stencilcraft/src/operations.rs (scoped #[allow(dead_code)] on OpDrag until 02-04)

key-decisions:
  - "PanelKey uses Member{layer, shape} for layer member leaves so N:M memberships keep unique row keys (plan sketched Shape(u64); extended for uniqueness)"
  - "Layer-row operation label/icon come from the OperationPreset catalogue where the operation matches a preset (D-17), keeping operations.rs off the dead-code lint without deleting 02-04/02-05's drag payload"
  - "Worktree changes left uncommitted per plan (02-02 convention); only this SUMMARY is committed in the planning repo"

patterns-established:
  - "build_layer_rows(doc, open) -> Vec<LayerRow> is the single source of tree structure; UNASSIGNED_GROUP sentinel expands the derived group"
  - "pass_number counts producing layers up to the index, so reorder renumbers automatically"

requirements-completed: [UI-05, UI-06, UI-07]

# Coverage metadata (#1602)
coverage:
  - id: D1
    description: "Pure layer-tree helpers (build_layer_rows, pass_number, effective_operation, membership_labels) with unit tests; Pass N skips container/Guide layers and renumbers after reorder; Unassigned lists exactly the zero-membership shapes and persists nothing"
    requirement: UI-06
    verification:
      - kind: unit
        ref: "crates/ui-egui/src/layers.rs#tests (pass_number_skips_container_and_guide_layers_and_renumbers_after_reorder, build_layer_rows_lists_exactly_the_unassigned_shapes_and_persists_nothing, effective_operation_resolves_the_owning_layer_and_none_for_unassigned, membership_labels_lists_every_owning_operation_layer_n_to_m) — cargo test -p stencilcraft-ui-egui --locked: 31 passed, 0 failed"
        status: pass
    human_judgment: false
  - id: D2
    description: "Canvas paint respects layer.visible and layer.color: shape_visual takes the owning &Layer, parsed #rrggbb overrides the operation token, unparseable colour falls back, kerf preview wins; shape.style is gone from ui-egui"
    requirement: UI-05
    verification:
      - kind: unit
        ref: "crates/ui-egui/src/lib.rs#tests::layer_colour_affects_visual, kerf_preview_changes_visuals — cargo test -p stencilcraft-ui-egui --locked"
        status: pass
      - kind: automated
        ref: "rg -n 'shape\.style' crates/ui-egui -> no matches"
        status: pass
    human_judgment: false
  - id: D3
    description: "Layers panel is a craft_ui List tree: layer group rows with eye/lock toggles dispatching LayerSetVisible/LayerSetLocked after show(), Pass N badges only on produces() layers, membership badges on object rows, colour swatch, collapsible derived Unassigned group"
    requirement: UI-05
    verification:
      - kind: automated
        ref: "cargo check -p stencilcraft --all-targets --locked -> exit 0; cargo clippy -p stencilcraft --all-targets --locked -- -D warnings -> clean"
        status: pass
    human_judgment: true
    rationale: "Tree rendering, toggle hit-targets and badge layout are visual/interactive; no headless test drives the egui List — verifier should click through the panel"
  - id: D4
    description: "Inspector shows the effective Operation and Source (layer name or Unassigned) for the panel's layer selection or the first selected shape, read-only, dispatching no commands"
    requirement: UI-07
    verification:
      - kind: unit
        ref: "crates/ui-egui/src/lib.rs#tests::effective_operation + layers.rs effective_operation test — cargo test -p stencilcraft-ui-egui --locked effective_operation: 2 passed"
        status: pass
      - kind: automated
        ref: "cargo check -p stencilcraft --all-targets --locked -> exit 0"
        status: pass
    human_judgment: true
    rationale: "Which row the inspector reflects depends on live panel selection state; verifier should select a layer and an object and read the rows"

# Metrics
duration: 28min
completed: 2026-11-24
status: complete
---

# Phase 02 Plan 03: Layers Tree Panel, Layer-Colour-Aware Paint, and Inspector Operation Summary

**Read-only nested Layers tree with Pass N and membership badges, a derived Unassigned group, layer-colour canvas paint, and an inspector effective-operation section — all panel values computed by tested pure helpers of the v2 document**

## Performance

- **Duration:** 28 min
- **Started:** 2026-11-24T19:50:00Z
- **Completed:** 2026-11-24T20:18:38Z
- **Tasks:** 3
- **Files modified:** 4 worktree files (uncommitted per plan) + this SUMMARY in the planning repo

## Accomplishments
- `crates/ui-egui/src/layers.rs` created: `pass_number`, `build_layer_rows`, `effective_operation`, `membership_labels` over `&Document`, with 4 unit tests covering Pass N renumbering, exact Unassigned membership, N:M labels, and unassigned `None`
- `shape_visual` now takes the owning `&Layer`; parsed `layer.color` overrides the Cut/Engrave token (neutral token otherwise), unparseable colours fall back, kerf preview unchanged; `shape.style` fully gone from ui-egui
- `layers_panel` rebuilt on `craft_ui::list::List` in tree mode: group rows with eye/lock toggles dispatching `LayerSetVisible`/`LayerSetLocked` after `.show()`, operation + `Pass N` detail, colour swatch, member leaves with membership badges, collapsible Unassigned group from `unassigned_shapes()`
- Inspector gained a read-only Operation section (Operation + Source rows) driven by `effective_operation`, showing the layer row's operation when it is the panel lead, else the first selected shape's source or "Unassigned"
- Plan-level gates green: `cargo test -p stencilcraft-ui-egui --locked` (31 passed), `cargo check -p stencilcraft --all-targets --locked`, clippy `-D warnings` on both crates, `cargo fmt --all -- --check`

## Task Commits

1. **Task 1: Pure layer-tree helpers in ui-egui with tests** — worktree changes uncommitted per plan; verified by `cargo test -p stencilcraft-ui-egui --locked` (31 ok) + clippy `-D warnings`
2. **Task 2: layers_panel as a craft_ui List tree** — worktree changes uncommitted per plan ("Do not commit"); verified by `cargo check -p stencilcraft --all-targets --locked` + clippy
3. **Task 3: Inspector effective-operation section** — worktree changes uncommitted per plan; verified by `cargo test -p stencilcraft-ui-egui --locked effective_operation` (2 ok) + `cargo check -p stencilcraft`

**Plan metadata:** this SUMMARY commit (rustfactor planning repo)

## Files Created/Modified
- worktree `crates/ui-egui/src/layers.rs` — created: `LayerRow`, `UNASSIGNED_GROUP`, four pure helpers, `mod tests`
- worktree `crates/ui-egui/src/lib.rs` — `pub mod layers`; `shape_visual(layer, …)`; paint passes the layer; tests `layer_colour_affects_visual`, `unassigned_rows`, `effective_operation`, `pass_number_skips_container_layers`
- worktree `apps/stencilcraft/src/main.rs` — `PanelKey`/`RowVisual`, `operation_label`/`operation_preset` (preset-catalogue backed), `open_layer_groups` + `layer_selection` fields, tree `layers_panel`, inspector Operation section
- worktree `apps/stencilcraft/src/operations.rs` — scoped `#[allow(dead_code)]` on `OpDrag` (drag wiring lands in 02-04)

## Decisions Made
- Member leaves keyed `PanelKey::Member{layer, shape}` so a shape in N layers yields N distinct rows (plan's `Shape(u64)` sketch would collide under D-02 N:M)
- Layer-row operation label/icon resolved through `OperationPreset` where the operation matches a built-in preset; other operations get a plain label fallback
- Tracer gate (Task 1) re-ran automated verify only — all `<automated>` checks passed, so expansion continued with no checkpoint (end-of-phase mode)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Pre-existing dead-code lint failures in `operations.rs` broke the plan's own clippy gate**
- **Found during:** Task 2 (clippy verify)
- **Issue:** 02-02's `OperationPreset`/`OpDrag` were unused, so `cargo clippy -p stencilcraft -- -D warnings` failed before Task 2's code could pass its acceptance criteria
- **Fix:** Layer rows now take label/icon from the preset catalogue (genuine use of `ALL`/`label`/`icon`/`operation`); `OpDrag` got a scoped `#[allow(dead_code)]` with a comment naming 02-04 as its consumer
- **Files modified:** apps/stencilcraft/src/main.rs, apps/stencilcraft/src/operations.rs
- **Verification:** `cargo clippy -p stencilcraft --all-targets --locked -- -D warnings` exit 0

**2. [Rule 1 - Bug] Test helper assumed an empty `Document::default()`**
- **Found during:** Task 1 (first test run)
- **Issue:** `Document::default()` starts with one operation-less `base` layer, shifting the test doc's layer indices
- **Fix:** Test helper uses the default base layer as index 0
- **Files modified:** crates/ui-egui/src/layers.rs (test-only)
- **Verification:** all 31 ui-egui tests pass

---

**Total deviations:** 2 auto-fixed (1 blocking lint gate, 1 test-only bug)
**Impact on plan:** No scope creep; both fixes keep the plan's own verification gates green. Worktree changes remain uncommitted per plan convention.

## Issues Encountered
- None beyond the deviations above.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- 02-04/02-05 can build reorder drag and the operation popup on `ListEvent::Reorder`/`OpDrag`, and assignment via `layer.add-members`/`layer.remove-members`; the tree, badges, and paint already reflect membership live
- Visual pass recommended: the tree panel and inspector section have no headless UI test (see coverage D3/D4)

---
*Phase: 02-layer-operations-ui-and-assignment*
*Completed: 2026-11-24*

## Self-Check: PASSED
- `crates/ui-egui/src/layers.rs` exists with the four pure helpers and a passing `mod tests`
- `rg -n "shape\.style" crates/ui-egui` -> no matches
- `cargo test -p stencilcraft-ui-egui --locked`: 31 passed, 0 failed (incl. the four new tests)
- `cargo check -p stencilcraft --all-targets --locked` exit 0; `cargo check --workspace --all-targets --locked` exit 0
- `cargo clippy -p stencilcraft -p stencilcraft-ui-egui --all-targets --locked -- -D warnings`: clean
- `cargo fmt --all -- --check`: clean
