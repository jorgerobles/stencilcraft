---
phase: 02-layer-operations-ui-and-assignment
plan: 04
subsystem: ui
tags: [rust, egui, craft-ui-menu-popup, opdrag-palette, send-to-layer, layer-add-members, layer-set-operation, ui-01, ui-02, ui-03]

# Dependency graph
requires:
  - phase: 02-layer-operations-ui-and-assignment/02-03
    provides: tree layers panel on craft_ui List with ListOutput.rows, OperationPreset catalogue, operation-less LayerAdd, green compile
provides:
  - Operation popup (configurable modifier + LMB on Add Layer, hammer-equivalent trailing button per layer row) dispatching LayerSetOperation
  - Draggable Operations palette with distinct OpDrag payload and drop hit-test onto layer rows only
  - Send to Layer… submenu with per-layer assign/remove entries dispatching LayerAddMembers/LayerRemoveMembers
  - Engine integration tests over the exact dot-named JSON (additive, idempotent, geometry-invariant, undoable)
affects: [02-05 reorder/panel polish, UI-01, UI-02, UI-03, production pass assignment]

# Actuals (#2632)
actuals:
  tokens: 12326
  tasks: 3
  commits: 1

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Popup pattern: Menu::new(&entries).show(ui, Popup::menu(&anchor).open_memory(cmd)) with the open command (Bool(true)) supplied only on the trigger frame; anchor Response persisted in app state so the popup survives frames"
    - "Drag routing: ui.dnd_drag_source with distinct OpDrag payload; on pointer release DragAndDrop::payload::<OpDrag> hit-tested against ListOutput.rows filtered to PanelKey::Layer only"
    - "Send to Layer… submenu built dynamically from doc.layers; locked layers disabled (T-02-13); never a synthetic Unassigned entry"

key-files:
  created: []
  modified:
    - worktree apps/stencilcraft/src/main.rs (op popup + palette + drop + TopMenuAction arms + OpPopupModifier pref)
    - worktree apps/stencilcraft/src/menus.rs (Send to Layer… submenu replacing todo!)
    - worktree apps/stencilcraft/src/operations.rs (OpDrag now used; dead_code allow removed)
    - worktree crates/engine/tests/production.rs (assignment_json_is_additive_idempotent_geometry_invariant_and_undoable)
    - worktree lint-only fixes unblocking the workspace clippy gate (doc/templates.rs, kerf, layout, svg, tiling, studio example)

key-decisions:
  - "No Hammer icon exists in the craft-ui icon catalogue at the pinned rev; used Icon::Settings as the row trailing button (closest semantic for 'set operation')"
  - "Shift+click on Add Layer adds the layer with operation: None first, then opens the popup targeting the new layer — D-01 and D-02 both hold"
  - "Popup modifier stored as UserPrefs.op_popup_modifier (OpPopupModifier enum, default Shift), not hardcoded"
  - "Worktree changes left uncommitted per plan convention; only this SUMMARY is committed in the planning repo"

patterns-established:
  - "Typed DragAndDrop::payload::<T> checks keep OpDrag drops distinct from ListDrag reorder payloads (T-02-11)"

requirements-completed: [UI-01, UI-02, UI-03]

# Coverage metadata (#1602)
coverage:
  - id: D1
    description: "Operation popup opens from shift+LMB on Add Layer and from the per-row trailing button, and choosing a preset dispatches LayerSetOperation visible on the row badge; plain Add Layer stays operation-less"
    requirement: UI-01
    verification:
      - kind: automated
        ref: "cargo check -p stencilcraft --all-targets --locked -> exit 0; cargo clippy -p stencilcraft --all-targets --locked -- -D warnings -> clean; rg 'LayerAdd' shows operation: None at both call sites"
        status: pass
    human_judgment: true
    rationale: "Popup anchoring, keyboard navigation and badge re-render are interactive egui behaviour with no headless driver — verifier should click through"
  - id: D2
    description: "Operations palette entries are draggable with OpDrag payloads and drop-onto-layer-row hit-tests ListOutput.rows, dispatching LayerSetOperation for the row under the pointer; shape/Unassigned rows ignored"
    requirement: UI-02
    verification:
      - kind: automated
        ref: "cargo clippy --workspace --all-targets --locked -- -D warnings -> clean (OpDrag used, no dead-code); cargo check -p stencilcraft --all-targets --locked -> exit 0"
        status: pass
    human_judgment: true
    rationale: "Drag-and-drop pointer choreography needs a real pointer — verifier should drag a palette entry onto a layer row"
  - id: D3
    description: "Send to Layer… submenu lists one assign and one remove entry per real layer (locked disabled, never Unassigned), dispatching LayerAddMembers/LayerRemoveMembers with the selection ids via run_command"
    requirement: UI-03
    verification:
      - kind: automated
        ref: "grep -v '^#' menus.rs | grep -c 'todo!(\"Send to Layer' -> 0; cargo check -p stencilcraft --all-targets --locked -> exit 0"
        status: pass
      - kind: integration
        ref: "crates/engine/tests/production.rs#assignment_json_is_additive_idempotent_geometry_invariant_and_undoable — cargo test -p stencilcraft-engine --locked: 8 passed in production.rs, 0 failed workspace-wide"
        status: pass
    human_judgment: true
    rationale: "Menu construction from live document state and enabled-state behaviour are best confirmed by opening the Object menu with a selection"
  - id: D4
    description: "Exact dot-named JSON (layer.add-members / layer.remove-members / layer.set-operation) is additive, idempotent, geometry-byte-equal, puts producing layers in production_sequence(), and undo restores prior state after every mutation"
    requirement: UI-03
    verification:
      - kind: integration
        ref: "crates/engine/tests/production.rs#assignment_json_is_additive_idempotent_geometry_invariant_and_undoable — pass; cargo test -p stencilcraft-engine --locked -> all suites ok (86+8+7+14+… passed, 0 failed)"
        status: pass
    human_judgment: false

# Metrics
duration: 11min
completed: 2026-11-24
status: complete
---

# Phase 02 Plan 04: Operation Popup, Draggable Palette, and Send to Layer… Assignment Summary

**Operation assignment via shift+click popup, per-row trailing button and draggable OpDrag palette, plus additive Send to Layer… context-menu assignment — all dispatched through the tested layer.* engine commands with geometry never touched**

## Performance

- **Duration:** 11 min
- **Started:** 2026-11-24T20:19:00Z
- **Completed:** 2026-11-24T20:30:25Z
- **Tasks:** 3
- **Files modified:** 4 worktree files this plan (uncommitted per plan) + 6 lint-only worktree files + this SUMMARY in the planning repo

## Accomplishments
- `layers_panel`: plain Add Layer dispatch keeps `operation: None` (D-01); the configured modifier (UserPrefs `op_popup_modifier`, default Shift) + click opens a `craft_ui::menu::Menu` popup built from `OperationPreset::ALL`, anchored to the button, dispatching `LayerSetOperation` for the new layer
- Hammer-equivalent trailing `IconButton` (Icon::Settings — no Hammer glyph in the pinned icon catalogue) on each layer row opens the same popup for that row index
- Operations palette section with `ui.dnd_drag_source` entries carrying the distinct `OpDrag(preset)` payload; on pointer release the payload is hit-tested against `ListOutput.rows`, accepting only `PanelKey::Layer` rects (T-02-11), dispatching `LayerSetOperation`
- `menus.rs`: `todo!("Send to Layer…")` replaced by a dynamic `Send to Layer…` submenu — one assign (`AssignToLayer { layer }`) and one `Remove from {name}` (`RemoveFromLayer { layer }`) entry per real layer, disabled for locked layers and when the selection is empty; both arms in `handle_top_menu_action` dispatch `LayerAddMembers`/`LayerRemoveMembers` with the selection ids through `run_command`
- `crates/engine/tests/production.rs`: new `assignment_json_is_additive_idempotent_geometry_invariant_and_undoable` proving the exact dot-named JSON is additive (original layer keeps membership), idempotent, geometry byte-equal, no-op on non-member remove, set/clear operation reflected in `production_sequence()`, and undo restores prior state after each mutation
- Plan-level gates green: `cargo check -p stencilcraft --all-targets --locked`, `cargo test -p stencilcraft-engine --locked` (all suites pass), `cargo clippy --workspace --all-targets --locked -- -D warnings`, `cargo fmt --all -- --check`

## Task Commits

1. **Task 1 (tracer): operation popup + row button + palette drag-onto-row** — worktree changes uncommitted per plan ("Do not commit"); tracer gate re-ran all `<automated>` verifies green (check, clippy, assign_preserves_geometry) and expanded with no checkpoint (end-of-phase mode)
2. **Task 2: Send to Layer… submenu with assign/remove** — worktree changes uncommitted per plan; verified by check + todo!-grep count 0 + engine tests
3. **Task 3 (TDD): end-to-end assignment tests over serde dot names** — worktree changes uncommitted per plan; `cargo test -p stencilcraft-engine --locked` 8/8 in production.rs

**Plan metadata:** this SUMMARY commit (rustfactor planning repo)

## Files Created/Modified
- worktree `apps/stencilcraft/src/main.rs` — `OpPopupModifier` pref, `op_popup` app state, footer shift+click popup, row trailing button, palette + drop hit-test, `AssignToLayer`/`RemoveFromLayer` arms
- worktree `apps/stencilcraft/src/menus.rs` — dynamic `Send to Layer…` submenu from `doc.layers`
- worktree `apps/stencilcraft/src/operations.rs` — `OpDrag` wired, `#[allow(dead_code)]` removed
- worktree `crates/engine/tests/production.rs` — new assignment JSON integration test
- worktree lint-only: `crates/doc/src/templates.rs`, `crates/kerf/src/lib.rs`, `crates/layout/src/lib.rs`, `crates/svg/src/lib.rs`, `crates/tiling/src/lib.rs`, `crates/studio/examples/studio_dump.rs`

## Decisions Made
- No Hammer icon in craft-ui at pinned rev 8504e84 → Icon::Settings trailing button (deviation from plan wording, same interaction)
- Popup anchor Response persisted in `StencilApp.op_popup`; `open_memory(Bool(true))` supplied only on the trigger frame so the stored clicked response cannot re-toggle the popup every frame
- Assignment dispatched only via `LayerAddMembers`/`LayerRemoveMembers`/`LayerSetOperation` through `DocumentCommand::run()`; no ShapesMoveToLayer/ShapesSetStyle

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Pre-existing workspace clippy failures blocked the plan's own verification gate**
- **Found during:** Task 3 (`cargo clippy --workspace --all-targets --locked -- -D warnings`)
- **Issue:** left by earlier waves' uncommitted work — `needless_lifetimes` on test helpers `member`/`member_mut` in doc/templates.rs, kerf, layout, svg, tiling; `needless_borrow` on two tiling test call sites; `unwrap_used` in the studio_dump example
- **Fix:** elided the lifetimes, removed the double borrows, replaced the example's `unwrap()` with `if let Some`
- **Files modified:** the six files listed above
- **Verification:** `cargo clippy --workspace --all-targets --locked -- -D warnings` exit 0

**2. [Rule 1 - Bug] New integration test assumed shapes land in layer 0 and that an empty producing layer is not a pass**
- **Found during:** Task 3 (first test run)
- **Issue:** `ShapeAddPath` places shapes in the style-matched layer, not necessarily layer 0; `production_sequence()` includes any `produces()` layer even with zero members
- **Fix:** assert additivity via `layer_of_shape(a)`; assert pass presence after `set-operation` and membership inside the pass after `add-members`
- **Files modified:** crates/engine/tests/production.rs (test-only)
- **Verification:** `cargo test -p stencilcraft-engine --locked` all suites pass

**3. [Rule 1 - Adaptation] Plan named a "hammer" icon that does not exist**
- **Found during:** Task 1
- **Issue:** craft-ui icon catalogue at rev 8504e84 has no Hammer/Wrench glyph
- **Fix:** Icon::Settings trailing button with "Set operation of {layer}" tooltip
- **Files modified:** apps/stencilcraft/src/main.rs
- **Verification:** clippy/check clean

---

**Total deviations:** 3 (1 blocking lint gate, 2 test/code adaptations)
**Impact on plan:** No scope creep; all fixes keep the plan's own verification gates green. Worktree changes remain uncommitted per plan convention.

## Issues Encountered
- egui `Popup::menu(&resp)` toggles on a clicked response, so replaying a stored click Response would flicker the popup — solved by supplying `open_memory(Bool(true))` only on the trigger frame
- `ListRow` trailing closures cannot capture `&mut` locals (FnMut escape), so the hammer click goes through a `RefCell`

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- 02-05 can build on the wired popup/palette/menu; the tree rows, badges and `ListOutput.rows` are the drop surface
- Visual pass recommended: popup anchoring, palette drag choreography, and menu enabled states have no headless UI test (coverage D1/D2/D3)

---
*Phase: 02-layer-operations-ui-and-assignment*
*Completed: 2026-11-24*

## Self-Check: PASSED
- `cargo check -p stencilcraft --all-targets --locked` exit 0
- `cargo test -p stencilcraft-engine --locked`: all suites ok (incl. `assignment_json_is_additive_idempotent_geometry_invariant_and_undoable` and `assign_preserves_geometry`), 0 failed
- `cargo clippy --workspace --all-targets --locked -- -D warnings` exit 0
- `cargo fmt --all -- --check` clean
- `grep -v '^#' apps/stencilcraft/src/menus.rs | grep -c 'todo!("Send to Layer'` -> 0
- Both `Command::LayerAdd` call sites pass `operation: None`; no `CutStyle::Cut` in any layer-add path
