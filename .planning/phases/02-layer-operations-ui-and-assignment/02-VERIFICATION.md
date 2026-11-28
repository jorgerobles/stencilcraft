---
phase: 02-layer-operations-ui-and-assignment
verified: 2026-11-24T21:00:00Z
status: human_needed
score: 7/7 must-haves verified
behavior_unverified: 5
overrides_applied: 0
behavior_unverified_items:
  - truth: "Operation popup opens from shift+click on Add Layer and per-row trailing button, dispatching LayerSetOperation (UI-01)"
    test: "Open StencilCraft, shift+click Add Layer, pick a preset from the popup; also click the settings icon on an existing layer row"
    expected: "Popup appears anchored to the button, choosing a preset sets the layer's operation badge; plain Add Layer creates an operation-less layer"
    why_human: "Popup anchoring, keyboard navigation, and badge re-render are interactive egui behavior with no headless driver"
  - truth: "Operations palette entries are draggable with OpDrag payloads and drop onto layer rows (UI-02)"
    test: "Drag an operation from the palette onto a layer row in the Layers panel"
    expected: "Layer's operation changes to the dragged preset; shape/Unassigned rows reject the drop"
    why_human: "Drag-and-drop pointer choreography needs a real pointer"
  - truth: "Send to Layer… submenu assigns/removes selection from layers (UI-03)"
    test: "Select objects, open Object menu → Send to Layer…, pick a layer to assign, then remove"
    expected: "Objects appear under the layer in the panel; removing them returns them to Unassigned if no other membership"
    why_human: "Menu construction from live document state and enabled-state behavior need visual confirmation"
  - truth: "Drag-handle layer reorder dispatches LayerMove with live Pass N renumbering (UI-04)"
    test: "Drag a layer row by its handle above another layer in the Layers panel"
    expected: "Layer order changes, Pass N badges renumber immediately"
    why_human: "Drag-handle pointer choreography and live badge renumbering need a real pointer"
  - truth: "Colour swatch opens picker, dispatches LayerSetColor; eye/lock toggles dispatch LayerSetVisible/LayerSetLocked (UI-05)"
    test: "Click a layer's colour swatch, pick a colour; toggle the eye icon and lock icon on a layer"
    expected: "Canvas shapes change to the picked colour; hidden layer's shapes disappear same frame; locked layer rejects assignment"
    why_human: "Popup opening, picker interaction, toggle click-through on canvas need a real UI session"
human_verification:
  - test: "Visual pass through the Layers panel and canvas"
    expected: "Layers tree renders with Pass N badges, membership badges, colour swatches, eye/lock toggles, and the derived Unassigned group; canvas paint reflects layer colour/visibility; inspector shows effective Operation + Source"
    why_human: "Headless environment has no display — all interactive egui behaviors (popup, drag, picker, toggle) need a GUI host"
---

# Phase 02: Layer/Operations UI and Assignment — Verification Report

**Phase Goal:** Users can build an ordered operation stack in StencilCraft and assign any selection of objects to it without changing geometry.
**Mode:** mvp
**Verified:** 2026-11-24T21:00:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths (Roadmap Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | User can open a Layers/Operations menu and add a layer by choosing an operation; no implicit Cut | ✓ VERIFIED | `apps/stencilcraft/src/main.rs:2861` (shift+click popup), `operations.rs:9-18` (preset catalogue), engine test `layer_add_is_operationless` + `document_new_has_no_cut` pass; D-01 honored (operation-less container, popup applies operation) |
| 2 | User can add/remove selected objects to/from a layer; geometry stays put | ✓ VERIFIED | `DocumentCommand::LayerAddMembers`/`LayerRemoveMembers` at `engine/src/document.rs:88-91`, `Send to Layer…` submenu at `menus.rs:215-232`, tests `assign_preserves_geometry` + `assign_is_additive_and_idempotent` + `assignment_json_is_additive_idempotent_geometry_invariant_and_undoable` all pass |
| 3 | User can reorder operation layers and see production order | ✓ VERIFIED | `ListEvent::Reorder` → `LayerMove` at `main.rs:3017-3019`, `pass_number` renumbering in `layers.rs:30`, tests `reorder_changes_pass_order` + `pass_number_renumbers_after_layer_move` pass |
| 4 | User can edit layer colour, settings, visibility, lock from Layers panel; canvas reflects changes | ✓ VERIFIED | Colour picker popup at `main.rs:3100-3139` dispatching `LayerSetColor`, eye/lock toggles dispatching `LayerSetVisible`/`LayerSetLocked`, paint respects `layer.visible`/`layer.color` (`shape_visual` takes `&Layer`), tests `layer_colour_affects_visual` + `hidden_layer_shapes_are_skipped_by_the_paint` + `layer_properties_roundtrip` pass |
| 5 | Layers panel shows operation/colour/visibility/lock/order; object rows show memberships; Unassigned group is derived not persisted | ✓ VERIFIED | `build_layer_rows` + `UNASSIGNED_GROUP` at `layers.rs:13,46-70`, `unassigned_shapes()` from v2 model, `membership_labels` at `layers.rs:80`, test `build_layer_rows_lists_exactly_the_unassigned_shapes_and_persists_nothing` passes |

### Requirements Coverage

| Requirement | Description | Status | Evidence |
|-------------|-------------|--------|----------|
| UI-01 | Layer/Operations menu + add-layer flow | ✓ SATISFIED | 02-02 (preset catalogue), 02-04 (popup + palette); `OperationPreset::ALL` = Cut/Engrave/Score/Guide |
| UI-02 | No silent Cut default | ✓ SATISFIED | 02-02: `DocumentNew` starts with operation-less base layer; both `LayerAdd` call sites pass `operation: None` |
| UI-03 | Add/remove objects from layer, geometry invariant | ✓ SATISFIED | 02-02: `LayerAddMembers`/`LayerRemoveMembers`; 02-04: `Send to Layer…` submenu; engine tests green |
| UI-04 | Reorder layers, see production order | ✓ SATISFIED | 02-01: `production_sequence()`; 02-05: drag reorder → `LayerMove`, Pass N badges |
| UI-05 | Edit colour/settings/visibility/lock from panel | ✓ SATISFIED | 02-03: eye/lock toggles; 02-05: colour picker → `LayerSetColor`; canvas paint respects layer properties |
| UI-06 | Object rows show memberships; Unassigned derived group | ✓ SATISFIED | 02-01: `unassigned_shapes()`; 02-03: `UNASSIGNED_GROUP` + `build_layer_rows` + `membership_labels` |
| UI-07 | Inspector shows effective operation + source | ✓ SATISFIED | 02-03: `effective_operation()` helper + inspector Operation section in `main.rs:3195` |

**Coverage: 7/7 UI requirements satisfied.**

### Plan SUMMARY Completeness

| Plan | SUMMARY Exists | Status |
|------|---------------|--------|
| 02-01 | ✓ | complete |
| 02-02 | ✓ | complete |
| 02-03 | ✓ | complete |
| 02-04 | ✓ | complete |
| 02-05 | ✓ | complete |

### Quality Gates

| Gate | Command | Result |
|------|---------|--------|
| Workspace compile | `cargo check --workspace --all-targets --locked` | ✓ exit 0 |
| Format | `cargo fmt --all -- --check` | ✓ clean |
| Lint | `cargo clippy --workspace --all-targets --locked -- -D warnings` | ✓ exit 0 |
| Engine tests | `cargo test -p stencilcraft-engine --locked` | ✓ 114 passed, 0 failed |
| UI tests | `cargo test -p stencilcraft-ui-egui --locked` | ✓ 33 passed, 0 failed |
| App tests | `cargo test -p stencilcraft --locked` | ✓ all passed, 0 failed |
| Layer rules | `cargo xtask layers` | ✓ ok |
| craft-libs pin | `git -C refs/craft-libs rev-parse phase-1-v2-model` | ✓ 4a2fada7cdc3c13d8db974752fae65419f36e920 |
| craft-libs tests | `cargo test -p craft-stencilcraft-format -p craft-stencilcraft-document --locked` | ✓ 67 passed, 0 failed |

### Anti-Patterns Found

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| None | No TBD/FIXME/XXX/todo! in phase-modified app source | — | Clean |

### Known Deferred Items

- 2 pre-existing `crates/import/tests/corpus.rs` failures (`dxf_round_trips_our_exporter`, `svg_registry_preserves_clipped_cut_lines_and_stroke_layers`) — logged in `deferred-items.md`, unrelated to Phase 2 UI work (import/dxf territory from Phase 1 migration).

### Human Verification Required

5 interactive UI behaviors need a GUI host to verify (popup anchoring, drag-and-drop, menu enabled states, colour picker, toggle click-through). All non-visual behaviors have automated test coverage.

### Gaps Summary

No blocking gaps. All 7 UI requirements have code-level evidence with passing automated tests. Phase goal is achieved at the code level; interactive UI confirmation requires a human with a display.

---

_Verified: 2026-11-24T21:00:00Z_
_Verifier: gsd-verifier_
