---
phase: 261126-dqo
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - apps/stencilcraft/specs/features/geometry_panel.feature
  - apps/stencilcraft/specs/features/operations_panel.feature
  - apps/stencilcraft/specs/features/profiles_panel.feature
  - apps/stencilcraft/specs/features/operation_assignment.feature
  - apps/stencilcraft/specs/features/bulk_operations.feature
  - apps/stencilcraft/specs/locators.md
  - apps/stencilcraft/specs/OPERATIONS-LAYERS-REQUIREMENTS.md
  - apps/stencilcraft/src/specs.rs
  - apps/stencilcraft/src/main.rs
  - apps/stencilcraft/src/operations.rs
  - crates/ui-egui/src/layers.rs
  - crates/ui-egui/src/lib.rs
autonomous: true
requirements: [OP-01, OP-02, OP-03, OP-04, OP-05, OP-06, LY-01, GEO-04, GEO-05, GEO-06, GEO-07, UI-03, UI-04, UI-05, UI-06]
user_setup: []

estimate:
  tokens: 62000
  raw_tokens: 46000
  tasks: 3
  confidence: low

must_haves:
  truths:
    - "Geometry rows have no operation-creation control, no operation ids in the row detail, and no view-only filter."
    - "Clicking a geometry operation pill opens only that operation's actions; the overflow opens one submenu per remaining operation."
    - "The Operations list exposes a visible lock toggle and double-clicking a member row selects that shape."
    - "The colour popup shows 40 palette swatches plus the custom hex/RGB controls and commits the chosen colour."
    - "Profiles lists Cut, Score and Engrave only; a new operation shows Profile Cut and editing settings shows Profile none."
    - "The plus chevron's appearance action opens a stroke/fill/both chooser; choosing a value selects all matching shapes and creates one operation over them."
  artifacts:
    - path: "apps/stencilcraft/specs/features/geometry_panel.feature"
      provides: "Contextual pill, no geometry creation, no operation-id/filter scenarios"
    - path: "apps/stencilcraft/specs/features/operations_panel.feature"
      provides: "Lock toggle, member double-click, palette, profile-link scenarios"
    - path: "apps/stencilcraft/specs/features/profiles_panel.feature"
      provides: "Base Cut/Score/Engrave profile scenarios"
    - path: "apps/stencilcraft/specs/features/bulk_operations.feature"
      provides: "Appearance popup scenarios"
    - path: "apps/stencilcraft/src/main.rs"
      provides: "Geometry/Operations panel implementation and app state"
    - path: "apps/stencilcraft/src/operations.rs"
      provides: "Profile catalogue"
    - path: "crates/ui-egui/src/lib.rs"
      provides: "40-colour operation palette"
    - path: "crates/ui-egui/src/layers.rs"
      provides: "Geometry row helpers without the view-only filter"
  key_links:
    - from: "apps/stencilcraft/src/main.rs"
      to: "crates/ui-egui/src/layers.rs"
      via: "operation_tags and build_geometry_rows"
    - from: "apps/stencilcraft/src/main.rs"
      to: "craft_ui::list::ToggleColumn::lock"
      via: "operations row lock toggle"
    - from: "apps/stencilcraft/src/main.rs"
      to: "craft_ui::controls::Swatch"
      via: "colour popup palette grid"
    - from: "apps/stencilcraft/src/main.rs"
      to: "craft_ui::menu::MenuItem::submenu"
      via: "geometry tag overflow submenus"
    - from: "apps/stencilcraft/src/main.rs"
      to: "crates/ui-egui/src/lib.rs"
      via: "OPERATION_PALETTE and operation_swatch_color"
---

<objective>
Deliver the Operations and Geometry UX wave: remove operation creation and the view-only filter from Geometry, make operation pills contextual, add a visible lock toggle and member double-click, add a 40-colour palette with the custom picker, introduce base Cut/Score/Engrave profiles with a provisional operation link, and replace by-colour bulk creation with an appearance popup.

Purpose: the current Geometry panel still creates operations from a per-row cog and hides shapes through a view-only filter, while the Operations panel hides locking and profiles and groups bulk creation by operation colour. These behaviours contradict the agreed contract and the locked user decisions.

Output: updated Gherkin specs and step vocabulary, a slimmer Geometry panel, an Operations panel with lock/member/profile behaviour, a palette-backed colour popup, and an appearance-based bulk chooser.
</objective>

<execution_context>
@/home/r2d2/.config/opencode/gsd-core/workflows/execute-plan.md
@/home/r2d2/.config/opencode/gsd-core/templates/summary.md
</execution_context>

<context>
@.planning/STATE.md
@AGENTS.md
@apps/stencilcraft/specs/OPERATIONS-LAYERS-REQUIREMENTS.md
@apps/stencilcraft/specs/features/geometry_panel.feature
@apps/stencilcraft/specs/features/operations_panel.feature
@apps/stencilcraft/specs/features/profiles_panel.feature
@apps/stencilcraft/specs/features/operation_assignment.feature
@apps/stencilcraft/specs/features/bulk_operations.feature
@apps/stencilcraft/specs/locators.md

Baseline: checkpoint commit `52fb11c`, worktree clean. Do not touch `refs/craft-libs`; consume only its public APIs (`craft_ui::controls::Swatch`, `craft_ui::list::ToggleColumn::lock`, `craft_ui::menu::MenuItem::submenu`). Do not add dependencies. Do not commit `.planning/` docs.
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Rewrite the Gherkin contract and step vocabulary first (RED)</name>
  <files>apps/stencilcraft/specs/features/geometry_panel.feature, apps/stencilcraft/specs/features/operations_panel.feature, apps/stencilcraft/specs/features/profiles_panel.feature, apps/stencilcraft/specs/features/operation_assignment.feature, apps/stencilcraft/specs/features/bulk_operations.feature, apps/stencilcraft/specs/locators.md, apps/stencilcraft/specs/OPERATIONS-LAYERS-REQUIREMENTS.md, apps/stencilcraft/src/specs.rs</files>
  <behavior>
    - New/changed scenarios describe the locked decisions and fail against the current UI for the right reason (missing widget or wrong document state), never with "undefined step".
    - Existing passing scenarios for nested groups, selection, grouping, assignment, and bulk thresholds keep passing.
  </behavior>
  <action>
    Rewrite the feature files to the locked contract before touching implementation. In `geometry_panel.feature`, keep the nested-group, shape-click, group-click and ungroup scenarios; replace the LY-01 hammer scenario with "operation tags are contextual pills" (Given an existing operation over shapes 1, 2, 3 named "cut"; click `Operation cut on #1`; assert `Remove #1 from cut` is on screen and `Operations of #1` is not); add "geometry offers no operation creation" (assert `New operation from outer` and `New operation from #1` are not on screen); add "geometry rows show no operation ids" (create a second operation over shape 1 and assert the row detail does not contain `#1 #1`); add "the tag overflow opens per-operation submenus" (create four operations over shape 1, click `More operations for #1`, and assert the fourth operation name is a menu item). In `operations_panel.feature`, replace every hammer setup with a Given operation over all shapes (reuse `a second operation` or add `an operation "{name}" over shapes {list}`); delete the hammer-creation scenario; add "the operations list has a visible lock toggle" (toggle `Lock Cut 2` and assert operation 2 is locked); add "double-clicking a member row selects that member" (expand operation 1, double-click `Member #2 of cut`, assert selected shapes are 2); change the settings scenario to assert `Kind` is on screen and `Show Cut 2 in geometry view` is not; add "the colour popup offers 40 palette colours" (assert `Palette colour 1`, `Palette colour 40` and `Hex` are on screen; click `Palette colour 1`, close, assert operation 2 colour `#000000`); add "a new operation links to Cut and editing settings unlinks it" (create with the plus, assert the operations panel shows `Profile Cut`; open settings, set power 40, close, assert `Profile none`). In `profiles_panel.feature`, replace the empty-stub scenario with "the profiles panel lists the base profiles": assert `Profile Cut`, `Profile Score`, `Profile Engrave` are on screen, `Profile Guide`, `No operation profiles yet` and `New profile` are not. In `operation_assignment.feature`, replace hammer setup with Given operations and replace the view-only filter scenario with "geometry has no per-operation filter" (open operation 1 settings and assert `Show cut in geometry view` is not on screen). In `bulk_operations.feature`, replace the by-colour scenario with "the plus menu offers an appearance popup": seed `shape 1 has stroke "#ff0000"` and `shape 2 has stroke "#00ff00"`, open `More new operation actions`, choose `New operation by appearance`, assert `Stroke`, `Fill` and `Both` are on screen, choose `Stroke #ff0000`, then assert selected shapes are 1, document has 2 operations, last operation has members 1. Add the matching steps to `specs.rs`: generic operation creation over a shape list, stroke/fill seeding through `ProductionCommand::SetProduction`, lock-toggle click, member-row double-click, palette-colour click, appearance-value click, and a MenuItem-role assertion for overflow submenus. Remove the hammer, view-filter, and old global-tag steps. Update `locators.md` (remove hammer/filter rows; add lock toggle, member double-click, palette swatch, profile row, appearance menu/window, contextual pill/overflow) and `OPERATIONS-LAYERS-REQUIREMENTS.md` (Decisions and spec map reflect the locked decisions; affected ledger rows become `wip`).
  </action>
  <verify>
    <automated>cargo fmt --all --check && (cargo test -p stencilcraft --bin stencilcraft gherkin -- --nocapture 2>&1 || true) | tee /tmp/261126-gherkin-red.log; ! grep -q "undefined step" /tmp/261126-gherkin-red.log && grep -q "baseline scenario(s) failed" /tmp/261126-gherkin-red.log</automated>
  </verify>
  <done>All feature files parse and run; no new step is undefined; the new/changed scenarios fail against the current UI (RED) while the retained baseline scenarios pass.</done>
</task>

<task type="auto">
  <name>Task 2: Remove Geometry creation/filter and add contextual pills, lock toggle and member double-click</name>
  <files>apps/stencilcraft/src/main.rs, crates/ui-egui/src/layers.rs</files>
  <action>
    In `main.rs`, delete the Geometry creation path: remove `pending_hammer`, the `hammer_click`/`hammer_targets` locals, the per-row `Icon::Settings` trailing button labelled `New operation from {label}`, and the `OperationCreate` dispatch block at the end of `geometry_panel`. Remove the view-only filter entirely: delete `hidden_ops` from `StencilApp` and its initializer, remove `hidden_ops` from `push_shape_row`, remove the `Filter`/`Unfilter` tag actions, remove `Show {name} in geometry view` from `operation_settings_popup`, and delete `shape_filtered_out` and its tests from `crates/ui-egui/src/layers.rs` (update the module doc comment). Change `RowVisual.pills` to carry `(op_id, name, fill)` and update every construction; in `push_shape_row` drop the `#opid` detail entirely, keep the shape label `#id`, and set `dimmed: false` for geometry rows. Add a `TagTarget` enum (`Operation(u64)`, `Overflow`) and change `tag_popup` to `(shape, TagTarget, Response)`. In the geometry row trailing cluster, a pill click stores `TagTarget::Operation(op.id)` and the overflow `More operations for #{id}` stores `TagTarget::Overflow`. Rewrite `tag_overflow_popup` so `TagTarget::Operation(id)` lists only that operation's Remove/Hide-Show/Select-members actions, and `TagTarget::Overflow` builds one `MenuItem::submenu(op.name, actions_for(op))` per operation after the first three pills; no flat all-operations list and no filter actions. In `operations_panel`, add `ToggleColumn::lock(format!("Lock {name}"), op.locked)` after the eye column and handle toggle column 1 as `OperationSetLocked`; keep column 0 as visibility. Handle `ListEvent::Activate(OpKey::Member { shape, .. })` by calling `set_selection(vec![shape])`. Remove the lock checkbox from the settings popover (the list toggle is the visible control) and keep kind/numeric settings. Update the unit tests in `main.rs`: delete the hammer-creation and view-filter tests; rebuild the tag-overflow test through the plus button; add tests for no geometry creation control, no operation-id detail, contextual pill menu, overflow submenu, lock toggle, member double-click, and absence of the view filter in settings.
  </action>
  <verify>
    <automated>cargo fmt --all --check && cargo clippy -p stencilcraft --all-targets -- -D warnings && cargo test -p stencilcraft --bin stencilcraft -- --skip gherkin</automated>
  </verify>
  <done>Geometry rows have no operation-creation control, no operation-id detail and no view-only filter; pills open per-operation actions; overflow opens per-operation submenus; the Operations list has a working lock toggle; double-clicking a member row selects that shape; all non-Gherkin unit tests pass.</done>
</task>

<task type="auto">
  <name>Task 3: Add base profiles, the 40-colour palette and the appearance bulk popup</name>
  <files>apps/stencilcraft/src/main.rs, apps/stencilcraft/src/operations.rs, crates/ui-egui/src/lib.rs</files>
  <action>
    Add `Profile` to `operations.rs`: variants `Cut`, `Score`, `Engrave`; `ALL`; `label`; and `for_kind` mapping only those three kinds (Guide and every other kind return `None`). Add `op_profiles: BTreeMap<u64, Option<Profile>>` to `StencilApp` (absent means derive from kind; `Some(None)` means explicitly unlinked; `Some(Some(profile))` means linked) and a `profile_of(&self, op)` helper that returns `None` for Guide, the explicit link when present, otherwise `Profile::for_kind(op.kind)`. After `run_bulk` creates operations, insert `Some(Profile::Cut)` for every newly created id. In `operation_settings_popup`, when the numeric settings changed and `OperationSetSettings` is dispatched, insert `Some(None)` for that id (editing settings unlinks); do not unlink on a pure kind change. Append `Profile {label}` or `Profile none` to the operations row detail. Replace the Profiles panel empty state with one row per `Profile::ALL` labelled `Profile Cut`, `Profile Score`, `Profile Engrave`; add a muted note that profile values are provisional; show no Guide and no `New profile`. Add `OPERATION_PALETTE: [&str; 40]` to `crates/ui-egui/src/lib.rs` with `#000000` first and placeholder entries after it. In `operation_color_popup`, render the 40 entries as `craft_ui::controls::Swatch` widgets labelled `Palette colour 1` through `Palette colour 40` in a wrapped grid; clicking one sets the working colour, hex text and dirty flag without closing the popup; keep the RGB sliders, Hex field, `Clear colour` and `Close`. Replace `BulkKind::ByColour` with `BulkKind::ByAppearance`: remove the operation-colour grouping from `plan`; add `AppearanceMode` (`Stroke`, `Fill`, `Both`) and `appearance_dialog`/`appearance_mode` state; make the chevron menu item `New operation by appearance` open the dialog instead of calling `request_bulk`. The dialog is a window titled `New operation by appearance` with `Stroke`, `Fill` and `Both` choices and a list of distinct existing values read from `Document.shapes` (`production.line_color`, `production.fill_color`, or both), capped at 100; each value is a button labelled `{mode} {value}`. Choosing one sets the selection to every matching shape id, creates one Cut operation over them, links it to `Profile::Cut`, and closes the dialog. Add unit tests for the profile catalogue/link lifecycle, the palette contents and the appearance grouping; keep the no-panic discipline (no `unwrap`/`expect`/`panic` outside tests).
  </action>
  <verify>
    <automated>cargo fmt --all --check && cargo clippy -p stencilcraft --all-targets -- -D warnings && cargo test -p stencilcraft --bin stencilcraft && cargo test -p stencilcraft --bin stencilcraft gherkin -- --nocapture</automated>
  </verify>
  <done>The Profiles panel lists Cut/Score/Engrave only; new operations show Profile Cut and editing settings shows Profile none; the colour popup offers 40 palette swatches plus the custom picker; the plus chevron offers an appearance popup that creates one operation from all matching shapes; all unit and Gherkin tests pass.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| user → colour popup | Free-text hex and RGB slider input reach `OperationSetColor`. |
| document → appearance popup | Shape line/fill values are document data rendered as chooser labels. |
| app → engine commands | UI actions dispatch `OperationCreate`, `OperationSetSettings`, `OperationSetLocked` and `OperationSetColor`. |

## STRIDE Threat Register

| Threat ID | Category | Component | Severity | Disposition | Mitigation Plan |
|-----------|----------|-----------|----------|-------------|-----------------|
| T-261126-01 | Tampering | `operation_color_popup` hex field | medium | mitigate | Parse with `stencilcraft_doc::canvas::parse_hex_color`; ignore invalid text; only canonical `#rrggbb` reaches `OperationSetColor`. |
| T-261126-02 | Denial of service | `appearance_values` list | low | mitigate | Cap distinct appearance values at 100 and skip shapes with no value for the selected mode; no unbounded widget tree. |
| T-261126-03 | Tampering | geometry pill actions | medium | mitigate | Remove actions dispatch existing `OperationRemoveMembers`/`OperationSetVisible`/selection commands by operation id; no new command surface. |
| T-261126-SC | Tampering | cargo installs | high | mitigate | No new dependencies; no package-legitimacy gate needed for this wave. |
</threat_model>

<verification>
Run from the checkout root:

- `cargo fmt --all --check`
- `cargo clippy -p stencilcraft --all-targets -- -D warnings`
- `cargo test -p stencilcraft --bin stencilcraft`
- `cargo test -p stencilcraft --bin stencilcraft gherkin -- --nocapture`

Expected: fmt clean, clippy clean with `-D warnings`, every unit test passes, and the Gherkin suite reports zero baseline failures with the new scenarios passing.
</verification>

<success_criteria>
- Geometry rows have no operation-creation cog, no operation ids in the row detail, and no view-only filter.
- Geometry pills open only the clicked operation's actions; the overflow opens one submenu per remaining operation.
- The Operations list shows a visible lock toggle that locks/unlocks through `OperationSetLocked`.
- Double-clicking a member row selects that shape.
- The colour popup offers 40 palette swatches plus the hex/RGB custom picker and commits the chosen colour.
- Profiles lists Cut, Score and Engrave only; a new operation shows Profile Cut and editing settings shows Profile none.
- The plus chevron's appearance action opens a stroke/fill/both chooser, lists existing appearance values from Geometry, and creates one operation over every matching shape.
- All four verification commands pass.
</success_criteria>

<output>
Create `.planning/quick/261126-dqo-operations-and-geometry-ux-wave-remove-g/261126-dqo-SUMMARY.md` when done.
</output>
