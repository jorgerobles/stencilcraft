# Phase 2: Layer/Operations UI and Assignment - Context

**Gathered:** 2026-11-24
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 2 delivers the StencilCraft UI for the operation-first model: a real Layer/Operations menu and panel, operation assignment to layers via popup/palette/drag, object-to-layer assignment, layer reordering with visible production order, and per-layer colour/visibility/lock/settings editing. It changes only the UI and engine command layer; geometry storage and the v2 document model from Phase 1 are untouched.

It does not emit per-operation G-code (Phase 3), job planning (Phase 4), or streaming/recovery/raster (next milestone).

</domain>

<decisions>
## Implementation Decisions

### Add-Layer Flow and Operation Picker

- **D-01:** Plain "Add Layer" creates an operation-less container layer (`operation: None`, per Phase 1 D-18); the operation is applied afterwards. The literal UI-01 reading ("add-layer must require an operation") was explicitly rejected by the user in favor of the Illustrator-appearance model. — **Reversibility:** reversible — undo only changes the add-layer default and dialog flow.
- **D-02:** Applying an operation to a layer happens through an operation popup opened by a **configurable shortcut (default: shift+LMB) on the Add Layer button** — NOT on layer rows, because shift+click on rows is reserved for multiple layer selection — plus a hammer/tool icon on the right side of the layer row that opens the same popup. The shortcut must be user-configurable in settings. — **Reversibility:** reversible — two entry points to the same dialog, shortcut binding is config.
- **D-03:** The popup lists the cooked operation presets, and an operations palette in the panel provides draggable operation entries that can be dropped onto a layer row to assign the operation. — **Reversibility:** reversible — UI-only choice of picker vs palette.
- **D-04:** Assigning an operation to a layer that already contains shapes includes all of them in that pass (consistent with Phase 1 D-03). — **Reversibility:** costly — undo changes assignment semantics shared with the document model.

### Assignment Interaction

- **D-05:** Objects are assigned to a layer via the context menu on the selection: "Send to Layer…" submenu listing layers (replaces the current `todo!("Send to Layer…")` in menus.rs). Works for one or many selected objects (UI-03). — **Reversibility:** reversible — menu structure only.
- **D-06:** Removal uses the same "Send to Layer…" submenu with a remove/unassign entry per layer, which strips the selection from that layer without touching geometry. — **Reversibility:** reversible — menu structure only.
- **D-07:** Assignment is additive: sending to a layer adds membership and keeps existing memberships in other operation layers (N:M per Phase 1 D-02). Never an exclusive move. — **Reversibility:** costly — undo breaks the engrave-then-cut multi-pass use case.
- **D-08:** Object rows in the Layers panel show membership badges naming each operation layer the object belongs to. — **Reversibility:** reversible — display only.

### Layers Panel Structure

- **D-09:** The panel is a nested tree: each layer row expands to show its member objects (with eye/lock toggles); a collapsible "Unassigned" group at the bottom lists exactly the zero-membership shapes, derived on the fly and never persisted (Phase 1 D-04, UI-06). — **Reversibility:** costly — undo restructures the whole layers panel in main.rs.
- **D-10:** Layer properties are edited inline on the row: eye (visibility), lock, colour swatch opening a colour picker, operation badge plus the hammer icon opening the operation popup. Row position doubles as order display. — **Reversibility:** reversible — control placement within the panel.

### Reordering and Production Order

- **D-11:** Layers are reordered by drag handles on the rows; row position is the order. — **Reversibility:** reversible — interaction choice.
- **D-12:** Production order is shown as a "Pass N" badge on each layer that has an operation (its position among operation layers, per Phase 1 D-19); operation-less layers show no pass badge. — **Reversibility:** reversible — display only.
- **D-13:** Drag interactions are unified: dragging an operation from the palette onto a row assigns the operation; dragging a row by its handle reorders it. — **Reversibility:** reversible — drag routing in the panel.

### Resolved During Research (2026-11-24)

- **D-14:** Phase 1's v2 document model is committed to a `storytold/craft-libs` feature branch and the worktree pins are bumped to that rev (format, document, raster-codecs, dxf); `Cargo.lock` regenerated. A local `[patch]` is fallback only. — **Reversibility:** reversible — pin revert.
- **D-15:** The ~178 v1→v2 call-site migration is Phase 2 **Wave 0**, with explicit per-crate tasks and a compile gate before any UI work. No separate prerequisite phase. — **Reversibility:** costly — splitting later means replanning.
- **D-16:** Additive membership logic (`add_member`/`remove_member`) lives in the **engine**, mutating the public v2 `Document.layers`/`shapes`; the craft-libs v2 model stays untouched this phase. — **Reversibility:** reversible — can promote helpers upstream later.
- **D-17:** Phase 2 ships a **minimal built-in operation-preset catalogue** (Cut/Engrave/Score/Guide, `OperationSettings::default()`, `preset_id` unset); no profiles/presets crate. — **Reversibility:** reversible — replace with profiles system later.

### Carried Forward from Phase 1 (already locked — do not re-discuss)

- Phase 1 D-02: geometry stored once; N:M memberships.
- Phase 1 D-03/D-04: operation on a layer includes its shapes; "Unassigned" is derived, never persisted.
- Phase 1 D-07: user-facing operations are cooked presets from the profiles system.
- Phase 1 D-18: new documents start with one operation-less base layer; no implicit Cut.
- Phase 1 D-19: order of `Some(operation)` layers is the production sequence.

### the agent's Discretion

- Exact egui widget composition (ListRow vs custom rows, drag-and-drop payload mechanics in egui).
- Where the operations palette lives (panel section vs flyout) as long as drag-onto-row works.
- Colour picker widget choice.
- Inspector (UI-07) effective-operation display details — not discussed; standard inspector pattern applies.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project and Phase

- `.planning/PROJECT.md` — project constraints, upstream rules (changes land in StencilCraft/craft-libs, not rustfactor).
- `.planning/REQUIREMENTS.md` — requirements UI-01..UI-07 assigned to this phase.
- `.planning/ROADMAP.md` — Phase 2 goal and success criteria.
- `.planning/phases/01-operation-first-document-model/01-CONTEXT.md` — locked Phase 1 model decisions (D-01..D-20) that this UI builds on.

### Existing UI Code (integration points)

- `worktrees/stencilcraft-planning/apps/stencilcraft/src/main.rs` — `layers_panel` (~line 2573): current flat panel, hardcoded `Command::LayerAdd { kind: CutStyle::Cut }` (~line 2672), eye/lock toggles.
- `worktrees/stencilcraft-planning/apps/stencilcraft/src/menus.rs` — `todo!("Send to Layer…")` (~line 236), SetStyle menu items (~line 356), Layers panel menu entry (~line 640).
- `worktrees/stencilcraft-planning/apps/stencilcraft/src/panels.rs`, `control.rs` — panel/control helpers.
- `worktrees/stencilcraft-planning/crates/engine/src/lib.rs` — `Command` enum incl. `ShapesSetStyle` (~line 146), `LayerAdd`/`LayerRemove`/`LayerSetVisible`.
- `worktrees/stencilcraft-planning/crates/engine/src/document.rs` — layer commands implementation (~line 2687).

### UI Component Library

- `craft_ui::list` (`ListRow`, `ToggleColumn::eye/lock`, `RowEvent::Toggle`, `EditEvent`), `Panel`/`PanelFooter`, `SectionHeader`, `IconButton`, `Icon`, `Inspector`/`InspectorHeader` — the reusable egui building blocks already used by the layers panel.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

- `craft_ui::list::ListRow` with `ToggleColumn::eye`/`lock`, `.selected()`, `.dimmed()`, row events — the basis for tree rows and membership badges.
- `Panel::new(...).footer_label(...).footer(...)` + `PanelFooter::show` + `IconButton` (Plus/Trash) — panel chrome for add/delete layer.
- `SectionHeader` — for the "Objects"/"Unassigned" sections.
- `Inspector`/`InspectorHeader` — for UI-07 effective-operation display.
- `Command` dispatch via `self.run_command(...)` + `retain_valid_selection()` — the pattern for all new layer/assignment commands.

### Established Patterns

- Layer rows are built from a snapshot `Vec<(usize, String, bool)>` then commands are dispatched after the UI pass — follow this for the tree.
- `can_delete = layers.len() > 1` guard for delete.
- Shape rows show `#id Cut/Engrave` labels derived from `s.style` — must become membership badges from the v2 model.

### Integration Points

- `main.rs:2672` — `Command::LayerAdd { kind: CutStyle::Cut }` hardcoded default; Phase 2 replaces with operation-less add.
- `menus.rs:236` — `todo!("Send to Layer…")`; Phase 2 implements the submenu.
- `menus.rs:356` — `SetStyle(CutStyle)` items; legacy style actions to be superseded by layer-operation assignment.
- `engine/src/lib.rs:1041` — `ShapesSetStyle` handler moves shapes between layers; assignment commands must not move geometry.

</code_context>

<specifics>
## Specific Ideas

- Illustrator-appearance model: a panel with draggable operations, applied to layers — user's explicit reference for the interaction design.
- Hammer/tool icon on the right of each layer row opening the same operation popup as shift+click.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 2-Layer/Operations UI and Assignment*
*Context gathered: 2026-11-24*
