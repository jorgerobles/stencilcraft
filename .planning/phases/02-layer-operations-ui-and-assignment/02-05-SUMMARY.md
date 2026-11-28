---
phase: 02-layer-operations-ui-and-assignment
plan: 05
subsystem: ui
tags: [rust, egui, craft-ui-list-reorder, layer-move, colour-picker, layer-set-color, ui-04, ui-05]

# Dependency graph
requires:
  - phase: 02-layer-operations-ui-and-assignment/02-04
    provides: layers panel tree on craft_ui List with ListOutput.rows, hammer/settings popup dispatching LayerSetOperation, OpDrag palette, green compile
provides:
  - Drag-handle reorder on layer rows (List::reorder + can_drag restricted to PanelKey::Layer) dispatching DocumentCommand::LayerMove after the UI pass, with per-frame Pass N renumbering
  - Interactive colour swatch per layer row opening an egui color_edit_button_srgba popup, dispatching canonical #rrggbb via LayerSetColor (and a clear path dispatching color: None)
  - ui-egui layer_swatch_color helper (stored colour, falling back to the operation token colour) and hidden-layer paint-skip test
  - pass_number renumbering-after-move test coverage
affects: [phase 2 verification, UI-04, UI-05, production pass order in the canvas]

# Actuals (#2632)
actuals:
  tokens: 15885
  tasks: 3
  commits: 1

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Reorder mapping: Move<PanelKey> { items, target, position } -> LayerMove { from, to } with to computed as the post-removal insert slot (Before/After target); accepts + can_drag closures restrict drags and drops to PanelKey::Layer rows only (D-11)"
    - "Colour popup pattern: anchor Response persisted in app state (color_popup), open_memory(SetOpenCommand::Bool(true)) supplied only on the trigger frame; working colour committed canonicalised via format_hex_color when the popup closes, Clear dispatches color: None"

key-files:
  created: []
  modified:
    - worktree apps/stencilcraft/src/main.rs (reorder wiring + Reorder arm, clickable swatch + colour popup, layer_rows_only/layer_move helpers, color_popup app state)
    - worktree crates/ui-egui/src/lib.rs (layer_swatch_color helper, hidden_layer_shapes_are_skipped_by_the_paint test)
    - worktree crates/ui-egui/src/layers.rs (pass_number_renumbers_after_layer_move test)

key-decisions:
  - "Reorder maps Move<PanelKey> to LayerMove with post-removal insert-slot semantics; single-layer drags only (items.len()==1), Into drops rejected"
  - "Colour is committed on popup close (one undo step) rather than per-frame while dragging the picker; canonicalised with format_hex_color, engine rejects unparseable strings (T-02-15)"
  - "pass_number renumber-after-move test applies the engine arm's exact remove/insert locally instead of adding stencilcraft-engine as a ui-egui dev-dep, which would force a Cargo.lock re-resolution (T-02-SC: no dependency changes); the through-the-engine path stays covered by engine's reorder_changes_pass_order"
  - "Worktree changes left uncommitted per plan convention ('Do not commit'); only this SUMMARY is committed in the planning repo"

patterns-established:
  - "can_drag + Reorder::accepts together keep member/shape/Unassigned rows out of reorder entirely (no drag starts, no drops accepted)"

requirements-completed: [UI-04, UI-05]

# Coverage metadata (#1602)
coverage:
  - id: D1
    description: "Layer rows reorder by drag handle; ListEvent::Reorder commits dispatch LayerMove through run_command after the UI pass and Pass N badges renumber next frame"
    requirement: UI-04
    verification:
      - kind: unit
        ref: "crates/ui-egui/src/layers.rs#pass_number_renumbers_after_layer_move — cargo test -p stencilcraft-ui-egui --locked pass_number: 3 passed"
        status: pass
      - kind: unit
        ref: "crates/engine/src/document.rs#reorder_changes_pass_order — cargo test -p stencilcraft-engine --locked: 1 passed"
        status: pass
      - kind: automated
        ref: "cargo check -p stencilcraft --all-targets --locked -> exit 0"
        status: pass
    human_judgment: true
    rationale: "Drag-handle pointer choreography and live badge renumbering on screen need a real pointer — verifier should drag a layer row and watch Pass N swap"
  - id: D2
    description: "Colour swatch per layer row opens an egui colour picker; chosen colour is canonicalised to #rrggbb and dispatched as LayerSetColor, clear path dispatches color: None, canvas reflects layer colour via shape_visual"
    requirement: UI-05
    verification:
      - kind: unit
        ref: "crates/ui-egui/src/lib.rs#layer_colour_affects_visual — cargo test -p stencilcraft-ui-egui --locked layer_colour_affects_visual: 1 passed"
        status: pass
      - kind: unit
        ref: "crates/engine/src/document.rs#layer_properties_roundtrip — cargo test -p stencilcraft-engine --locked: 1 passed"
        status: pass
    human_judgment: true
    rationale: "Popup opening, picker interaction and swatch re-render are interactive egui behaviour with no headless driver — verifier should click a swatch and pick a colour"
  - id: D3
    description: "Eye/lock ToggleColumns dispatch LayerSetVisible/LayerSetLocked through run_command and the paint loop skips hidden layers"
    requirement: UI-05
    verification:
      - kind: unit
        ref: "crates/ui-egui/src/lib.rs#hidden_layer_shapes_are_skipped_by_the_paint — cargo test -p stencilcraft-ui-egui --locked hidden_layer: 1 passed"
        status: pass
      - kind: automated
        ref: "cargo clippy --workspace --all-targets --locked -- -D warnings -> exit 0; cargo fmt --all -- --check -> clean; cargo xtask layers -> ok"
        status: pass
    human_judgment: true
    rationale: "Toggle click-through on canvas needs a real UI session — verifier should toggle the eye and watch shapes disappear same frame"

# Metrics
duration: 22min
completed: 2026-11-24
status: complete
---

# Phase 02 Plan 05: Layer Reorder, Colour Picker, and Property Editing Summary

**Drag-handle layer reorder wired to `LayerMove` with live Pass N badges, an egui colour-picker swatch dispatching canonical `LayerSetColor`, and tested visibility/lock/paint gating — all mutations through `run_command`**

## Performance

- **Duration:** 22 min
- **Started:** 2026-11-24T20:19:00Z
- **Completed:** 2026-11-24T20:41:36Z
- **Tasks:** 3
- **Files modified:** 3 worktree files this plan (uncommitted per plan) + this SUMMARY in the planning repo

## Accomplishments
- `layers_panel` `List` now has `.can_drag(|key| matches!(key, PanelKey::Layer(_)))` and `.reorder(Reorder::new(Axis::Vertical).accepts(layer_rows_only))`; committed `ListEvent::Reorder(EditEvent::Commit { value, .. })` gestures map through `layer_move()` (Before/After target → post-removal insert slot) and dispatch `DocumentCommand::LayerMove { from, to }` via `run_command` after the UI pass — Pass N badges renumber next frame via the per-frame `pass_number`
- Colour swatch is now a clickable trailing button on layer rows opening an egui `color_edit_button_srgba` popup; on close the picked colour is canonicalised with `format_hex_color` and dispatched as `LayerSetColor { color: Some(hex) }`, and a "Clear colour" button dispatches `color: None`; the swatch itself shows `layer_swatch_color` (stored colour, else operation token colour)
- Eye/lock toggles (already wired in 02-03) dispatch `LayerSetVisible`/`LayerSetLocked`; new ui-egui test `hidden_layer_shapes_are_skipped_by_the_paint` proves the paint loop skips hidden layers
- New ui-egui test `pass_number_renumbers_after_layer_move` covers renumbering after the engine arm's exact remove/insert move; engine `reorder_changes_pass_order` covers the through-the-engine path
- Gates green: `cargo check -p stencilcraft --all-targets --locked`, `cargo clippy --workspace --all-targets --locked -- -D warnings`, `cargo fmt --all -- --check`, `cargo xtask layers`, and `cargo test -p stencilcraft-engine -p stencilcraft-ui-egui -p stencilcraft --locked` (all suites pass)

## Task Commits

1. **Task 1 (tracer): drag-handle reorder → LayerMove with live Pass N** — worktree changes uncommitted per plan ("Do not commit"); tracer gate re-ran all `<automated>` verifies green (pass_number tests, reorder_changes_pass_order, cargo check) and expanded with no checkpoint
2. **Task 2: colour swatch picker + canonical LayerSetColor** — worktree changes uncommitted per plan; verified by layer_colour_affects_visual + layer_properties_roundtrip + cargo check
3. **Task 3: visibility/lock/settings confirmation + paint-skip test + phase gate** — worktree changes uncommitted per plan; hidden_layer test passes, fmt/clippy/layers green

**Plan metadata:** this SUMMARY commit (rustfactor planning repo)

## Files Created/Modified
- worktree `apps/stencilcraft/src/main.rs` — `layer_rows_only`/`layer_move` helpers, `can_drag`+`.reorder` on the layers List, `ListEvent::Reorder` arm dispatching `LayerMove`, clickable swatch + colour picker popup (`color_popup`/`color_popup_value`/`color_popup_dirty` app state), `layer_swatch_color` swatch source
- worktree `crates/ui-egui/src/lib.rs` — `pub fn layer_swatch_color`, `hidden_layer_shapes_are_skipped_by_the_paint` test
- worktree `crates/ui-egui/src/layers.rs` — `pass_number_renumbers_after_layer_move` test

## Decisions Made
- `Move<PanelKey>` → `LayerMove` uses post-removal insert-slot semantics; only single-layer drags, `Into` drops and Alt-copies are rejected by `layer_rows_only` (D-11: row position is the order, layer rows only)
- Colour commits once on popup close (single undo step) rather than per picker frame; canonical #rrggbb via `format_hex_color` (T-02-15)
- No new dependency edges: the ui-egui renumber test applies the engine arm's remove/insert locally because adding `stencilcraft-engine` as a dev-dep forces a workspace-wide Cargo.lock re-resolution (T-02-SC)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Adding stencilcraft-engine as a ui-egui dev-dependency would churn Cargo.lock workspace-wide**
- **Found during:** Task 1 (pass_number test placement)
- **Issue:** any workspace manifest change triggers full re-resolution, rewriting patched git sources/versions across the lock — a dependency change the plan's threat register forbids (T-02-SC)
- **Fix:** test applies the engine `layer.move` arm's exact `remove(from); insert(to)` locally; through-the-engine coverage remains `reorder_changes_pass_order` (passes)
- **Files modified:** crates/ui-egui/src/layers.rs (test-only)
- **Verification:** `cargo test -p stencilcraft-ui-egui --locked pass_number` 3 passed; Cargo.lock unchanged vs its pre-plan state

**2. [Rule 1 - Bug] Colour-popup compile fixes against the pinned egui fork**
- **Found during:** Task 2 (first cargo check)
- **Issue:** this egui fork's `Popup::show` takes only the content closure (no `ui` arg); `Response` is Clone but not Copy; `PathShape.stroke.color` is `ColorMode` while `LineSegment.stroke.color` is `Color32`
- **Fix:** adjusted call sites and test match arms accordingly
- **Files modified:** apps/stencilcraft/src/main.rs, crates/ui-egui/src/lib.rs
- **Verification:** `cargo check -p stencilcraft --all-targets --locked` exit 0; clippy -D warnings clean

---

**Total deviations:** 2 (1 dependency-scope guard, 1 API adaptation). No scope creep.
**Impact on plan:** plan's must-haves all hold; the only unmet acceptance row is the workspace-wide `cargo xtask ci` due to pre-existing failures below.

## Issues Encountered
- `cargo xtask ci` is NOT fully green: `crates/import/tests/corpus.rs::dxf_round_trips_our_exporter` and `::svg_registry_preserves_clipped_cut_lines_and_stroke_layers` fail — both already logged in this phase's `deferred-items.md` from 02-02 (import/dxf territory, untouched by this plan; the import crate cannot depend on ui-egui/app per `cargo xtask layers`). All other CI stages (fmt, clippy -D warnings, layers, workspace tests up to that point) are green, and `cargo test -p stencilcraft-engine -p stencilcraft-ui-egui -p stencilcraft --locked` is fully green (63+86+7+14+8+33 passed, 0 failed).
- Accidental `git checkout Cargo.lock` mid-run discarded the worktree's pre-existing uncommitted lock update; regenerated consistently with `cargo check --offline` and re-verified `--locked` passes. No version changes introduced by this plan.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- UI-04/UI-05 interactions are wired and unit-covered; a visual pass is recommended for drag choreography, the colour popup, and toggle click-through (coverage D1/D2/D3 are human_judgment)
- The two pre-existing import-corpus failures remain the phase's open deferred items for a follow-up import/dxf wave

---
*Phase: 02-layer-operations-ui-and-assignment*
*Completed: 2026-11-24*

## Self-Check: PASSED
- `cargo check -p stencilcraft --all-targets --locked` exit 0
- `cargo test -p stencilcraft-ui-egui --locked pass_number` → 3 passed; `layer_colour_affects_visual` → 1 passed; `hidden_layer_shapes_are_skipped_by_the_paint` → 1 passed
- `cargo test -p stencilcraft-engine --locked reorder_changes_pass_order` → 1 passed; `layer_properties_roundtrip` → 1 passed
- `cargo clippy --workspace --all-targets --locked -- -D warnings` exit 0; `cargo fmt --all -- --check` clean; `cargo xtask layers` ok
- `cargo test -p stencilcraft-engine -p stencilcraft-ui-egui -p stencilcraft --locked` all suites ok, 0 failed
- Known exception: 2 pre-existing `crates/import` corpus failures (deferred, see deferred-items.md) — unrelated to this plan's files
