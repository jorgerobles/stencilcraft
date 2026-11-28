---
phase: 261127-pfl
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - apps/stencilcraft/specs/features/operations_panel.feature
  - apps/stencilcraft/specs/features/profiles_panel.feature
  - apps/stencilcraft/specs/locators.md
  - apps/stencilcraft/specs/OPERATIONS-LAYERS-REQUIREMENTS.md
  - apps/stencilcraft/specs/GAPS.md
  - apps/stencilcraft/src/specs.rs
  - apps/stencilcraft/src/main.rs
  - crates/engine/src/document.rs
  - ROADMAP.md
autonomous: true
requirements: [OP-03, OP-07]
user_setup: []

estimate:
  tokens: 70000
  raw_tokens: 52000
  tasks: 3
  confidence: low

must_haves:
  truths:
    - "Operation rows render [kind thumb][link chain][profile swatch]; the chain is dimmed when linked and reports LinkToggled to break the link."
    - "A linked operation shows the profile colour; an unlinked one shows its own colour at 50% opacity."
    - "Dragging a member row onto another operation moves it; Shift-drag adds it; the cursor is a plus while Shift is held; one undo step."
    - "The colour palette uses 10 px cells, a 1 px gap and a 1 px hover/selected ring; all 40 swatches remain."
    - "Profile rows show a link+count column, a swatch, the name button, a description column and the cog."
    - "Extract/Import/Export are footer icon buttons (eyedropper, folder-open/arrow-square-out) with tooltips."
    - "Applying a profile to operations with different settings asks for confirmation."
  artifacts:
    - path: "apps/stencilcraft/src/main.rs"
      provides: "Profile id/color, derived matching/count, link glyph, member drag, compact palette, profile columns and warning dialog"
    - path: "crates/engine/src/document.rs"
      provides: "Atomic OperationMoveMembers command"
    - path: "apps/stencilcraft/specs/features/operations_panel.feature"
      provides: "Link state, swatch and member-drag scenarios"
    - path: "apps/stencilcraft/specs/features/profiles_panel.feature"
      provides: "Columns, counts, footer icons and warning scenarios"
  key_links:
    - from: "apps/stencilcraft/src/main.rs"
      to: "craft_ui::list::ListRow::thumb_link"
      via: "operation row link glyph and RowEvent::LinkToggled"
    - from: "apps/stencilcraft/src/main.rs"
      to: "crates/engine/src/document.rs"
      via: "OperationMoveMembers on a member-row drop"
---

<objective>
Implement the formalized Profiles link UX wave without touching the craft-libs pin.

Purpose: the current link is a right-side button, the profile rows mix name and description,
the palette is loose, and members cannot move between operations. The agreed Photoshop-style
model puts a dimmed chain between the operation and its profile swatch, keeps the link
derived from kind+settings, and makes the Profiles panel a scannable library.

Output: extended `Profile` (id/color), operation rows with the chain+swatch cluster, member
drag move/Shift-add, a compact palette, profile columns with counts, footer icon actions, and
a differing-settings confirmation. Specs are RED first, then green.
</objective>

<execution_context>
@/home/r2d2/.config/opencode/gsd-core/workflows/execute-plan.md
@/home/r2d2/.config/opencode/gsd-core/templates/summary.md
</execution_context>

<context>
@.planning/STATE.md
@AGENTS.md
@.planning/quick/261127-pfl-profiles-link-ux-wave/261127-pfl-CONTEXT.md
@apps/stencilcraft/specs/OPERATIONS-LAYERS-REQUIREMENTS.md
@apps/stencilcraft/specs/features/operations_panel.feature
@apps/stencilcraft/specs/features/profiles_panel.feature
@apps/stencilcraft/specs/locators.md

Baseline: `e73c872` (42 baseline scenarios, 76 unit tests). Do not touch `refs/craft-libs`
or `Cargo.lock`; consume only public APIs. Existing link = derived kind+settings plus
`StencilApp.detached_ops`; keep that model (see CONTEXT).
</context>

<tasks>
<task type="auto" tdd="true">
  <name>Task 1: RED specs and the Profile model</name>
  <files>apps/stencilcraft/specs/features/operations_panel.feature, apps/stencilcraft/specs/features/profiles_panel.feature, apps/stencilcraft/specs/locators.md, apps/stencilcraft/src/specs.rs, apps/stencilcraft/src/main.rs</files>
  <action>
Add the failing scenarios and step vocabulary before the UI:
- operations: "a linked operation shows the profile swatch and chain" (link Score, assert the chain label "Unlink Profile Score" and the swatch label "Profile colour Score"); "breaking the link removes the chain" (click the chain, assert "Link Profile Score" appears and the row shows Profile none); "dragging a member row moves it" (op 1 over 1,2,3 and op 2 over 1; drag "#3" from op 1 onto op 2; assert op 1 members 1,2 and op 2 members 1,3); "shift-dragging a member adds it" (same setup, Shift held; assert op 1 still has 1,2,3 and op 2 has 1,3).
- profiles: "a profile row shows the linked count" (link two operations to Score, assert "2 linked to Profile Score" is on screen); "the profile description is a separate column" (assert the row button "Profile Score" and the description text are distinct nodes); "applying a profile with different settings asks first" (op 2 power 40, select it, click "Profile Score", assert "1 operation differs. Link anyway?" is on screen; click "Link anyway", assert kind score).
Add the harness steps: a drag that can hold Shift (extend `drag_row_containing` with modifiers), a step to assert a label is on screen, and the confirm dialog buttons. Keep every new scenario failing for the right reason, never "undefined step".
Then extend `Profile` with `id: u64` and `color: String` (serde defaults), give the base profiles stable ids 1..3 and distinct colours, have `extract_profiles` inherit the operation colour and assign `max(id)+1`, and add pure helpers `profile_matches(op, profile)`, `linked_count(profiles, ops, detached, index)`, and `linked_profile_color(op, ...)`. Add unit tests for id assignment, colour inheritance and count/detach.
  </action>
  <verify>
    <automated>cargo fmt --all --check && (cargo test -p stencilcraft --bin stencilcraft gherkin -- --nocapture 2>&1 || true) | tee /tmp/261127-gherkin-red.log; ! grep -q "undefined step" /tmp/261127-gherkin-red.log && grep -q "baseline scenario(s) failed" /tmp/261127-gherkin-red.log</automated>
  </verify>
  <done>Feature files parse and run; no undefined steps; the new scenarios fail against the current UI (RED) while the retained 42 baseline scenarios pass.</done>
</task>

<task type="auto">
  <name>Task 2: Operations row link cluster, compact palette and member drag</name>
  <files>apps/stencilcraft/src/main.rs, crates/engine/src/document.rs</files>
  <action>
Operations rows: use `ListRow::thumb` for the kind icon and `extra_thumb` for the profile swatch, then `thumb_link(linked, "Unlink Profile {name}")` so the chain sits between them (Photoshop's position). Handle `ListEvent::Row(RowEvent::LinkToggled)` by inserting the operation id into `detached_ops` (break) or re-applying the profile (link). Paint the swatch from the linked profile colour; for unlinked operations paint the operation colour at 50% alpha. Keep the member trash slots aligned.
Palette: `main.rs` `operation_color_popup` grid becomes 10 px cells, `spacing([1.0, 1.0])`, and a 1 px stroke on hover/selected (draw inside the cell so neighbours do not shift).
Member drag: allow `OpKey::Member` in `List::can_drag`; on release over an `OpKey::Op` row, group the dragged members by source op. Shift adds only; a normal drag moves (add to target + remove from source) through a new atomic `DocumentCommand::OperationMoveMembers { from, to, ids }` in `crates/engine/src/document.rs` (one undo step; dedupe; ignore same-source). Set `CursorIcon::Copy` while Shift is held during the drag. Locked target or no target reports a status and changes nothing. Add engine unit tests for move, Shift-copy semantics (add only), and dedupe.
  </action>
  <verify>
    <automated>cargo fmt --all --check && cargo test -p stencilcraft && cargo test -p stencilcraft-engine && cargo clippy -p stencilcraft --all-targets -- -D warnings</automated>
  </verify>
  <done>The link cluster, swatch colours, 1 px palette and member move/add pass their scenarios; the engine move command is atomic and tested; clippy clean.</done>
</task>

<task type="auto">
  <name>Task 3: Profiles columns, footer icons, warning and docs</name>
  <files>apps/stencilcraft/src/main.rs, apps/stencilcraft/specs/OPERATIONS-LAYERS-REQUIREMENTS.md, apps/stencilcraft/specs/locators.md, apps/stencilcraft/specs/GAPS.md, ROADMAP.md</files>
  <action>
Rewrite the profile rows as a fixed grid: `[chain + N] [swatch] [button name] [description column] [cog]`. The chain is a read-only dimmed indicator (paint it in the leading thumb) with the count of operations in the current document linked to that profile; the name button applies the profile to the selection. Keep the cog popup. Move Extract/Import/Export to footer icon buttons: `Icon::Eyedropper` (Extract), `Icon::FolderOpen` (Import), `Icon::Open` (Export), each with a tooltip and accessible name. When applying a profile, count the selected operations whose kind/settings differ; if any, open a confirm dialog ("N operation(s) differ. Link anyway?" / Cancel) and only apply on confirm. Record the `arrow-square-in` substitution and the session-only detach in `GAPS.md`; update OP-03/OP-07, `locators.md` and the ROADMAP entry.
  </action>
  <verify>
    <automated>cargo fmt --all --check && SPEC_INCLUDE_GAPS=1 cargo test -p stencilcraft --bin stencilcraft specs -- --nocapture 2>&1 | grep -E "gherkin: .*0 failed" && cargo test -p stencilcraft && cargo clippy -p stencilcraft --all-targets -- -D warnings</automated>
  </verify>
  <done>Profile columns, counts, footer icons and the differing-settings confirmation pass; all specs and unit tests green; docs and GAPS updated.</done>
</task>
</tasks>

<verification>
- `cargo test -p stencilcraft` (unit + harness)
- `cargo test -p stencilcraft-engine`
- `SPEC_INCLUDE_GAPS=1 cargo test -p stencilcraft --bin stencilcraft specs -- --nocapture` → `0 failed, 0 gaps`
- `cargo fmt --all --check`
- `cargo clippy -p stencilcraft --all-targets -- -D warnings`
- `cargo xtask layers`
</verification>
