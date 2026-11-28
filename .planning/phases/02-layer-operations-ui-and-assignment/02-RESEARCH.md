# Phase 2: Layer/Operations UI and Assignment — Research

**Researched:** 2026-11-24
**Domain:** Rust / egui desktop UI on top of the Phase 1 operation-first document model (StencilCraft + craft-libs)
**Confidence:** MEDIUM (UI APIs verified against the exact pinned craft-ui rev; the biggest unknown is a cross-repo build/integration prerequisite that the phase framing understates)

## Summary

Phase 2 builds the StencilCraft Layers/Operations UI and assignment commands on top of the Phase 1 v2 document model. The UI building blocks exist in `craft-ui` at the worktree's pinned rev (`8504e84…`): `List` (tree mode, selection, drag-reorder, foreign-payload drop rects), `ListRow` (eye/lock toggles, trailing items, detail line), `TreeRow`/`RowInfo`, `Inspector`, `Menu`, `Panel`/`PanelFooter`, `SectionHeader`. The v2 model already exposes the three queries the panel needs — `Layer.members`, `Document::unassigned_shapes()` and `Document::production_sequence()` — and the engine already has `LayerMove`, `LayerSetColor`, `LayerSetLocked`, `LayerRename`, `LayerSetVisible`; what is missing is additive membership commands, an operation-set command, and UI wiring.

**The dominant risk is not the UI — it is the build boundary.** The worktree pins `craft-stencilcraft-document`/`-format` at rev `75c70fa6…`, which is the **v1** model (`Layer.kind`/`Layer.shapes`, `Shape.style`). Phase 1's v2 model exists only as **uncommitted working-tree changes** in `refs/craft-libs` (HEAD `249a41b`). The worktree source (engine, ui-egui, gcode, layout, apps, mcp, studio, dxf) is written against v1 and has ~178 non-test references to the removed fields (prior-session grep: engine 111, ui-egui 12, layout 16, apps/stencilcraft 23, stencilcraft-cli 6, gcode 4, mcp 3, studio 3; recount in Wave 0). Phase 2 cannot compile against Phase 1 until the worktree is re-pinned and those call sites are migrated. This is explicit in the Phase 1 research: "The StencilCraft `[patch]`/branch wiring is Phase 2 dev setup, not a Phase 1 task."

**Primary recommendation:** Treat Phase 2 as (Wave 0) dev-setup + mechanical v1→v2 migration of the worktree, then (Waves 1–4) engine membership/operation commands with geometry-invariance tests, then the Layers tree, palette/popup assignment, reorder/Pass-N, and inspector. Get a user decision on the migration scope and the pin strategy before planning tasks.

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Plain "Add Layer" creates an operation-less container layer (`operation: None`, per Phase 1 D-18); the operation is applied afterwards. The literal UI-01 reading ("add-layer must require an operation") was explicitly rejected by the user in favor of the Illustrator-appearance model. — **Reversibility:** reversible — undo only changes the add-layer default and dialog flow.
- **D-02:** Applying an operation to a layer happens through an operation popup opened by a **configurable shortcut (default: shift+LMB) on the Add Layer button** — NOT on layer rows, because shift+click on rows is reserved for multiple layer selection — plus a hammer/tool icon on the right side of the layer row that opens the same popup. The shortcut must be user-configurable in settings. — **Reversibility:** reversible — two entry points to the same dialog, shortcut binding is config.
- **D-03:** The popup lists the cooked operation presets, and an operations palette in the panel provides draggable operation entries that can be dropped onto a layer row to assign the operation. — **Reversibility:** reversible — UI-only choice of picker vs palette.
- **D-04:** Assigning an operation to a layer that already contains shapes includes all of them in that pass (consistent with Phase 1 D-03). — **Reversibility:** costly — undo changes assignment semantics shared with the document model.
- **D-05:** Objects are assigned to a layer via the context menu on the selection: "Send to Layer…" submenu listing layers (replaces the current `todo!("Send to Layer…")` in menus.rs). Works for one or many selected objects (UI-03). — **Reversibility:** reversible — menu structure only.
- **D-06:** Removal uses the same "Send to Layer…" submenu with a remove/unassign entry per layer, which strips the selection from that layer without touching geometry. — **Reversibility:** reversible — menu structure only.
- **D-07:** Assignment is additive: sending to a layer adds membership and keeps existing memberships in other operation layers (N:M per Phase 1 D-02). Never an exclusive move. — **Reversibility:** costly — undo breaks the engrave-then-cut multi-pass use case.
- **D-08:** Object rows in the Layers panel show membership badges naming each operation layer the object belongs to. — **Reversibility:** reversible — display only.
- **D-09:** The panel is a nested tree: each layer row expands to show its member objects (with eye/lock toggles); a collapsible "Unassigned" group at the bottom lists exactly the zero-membership shapes, derived on the fly and never persisted (Phase 1 D-04, UI-06). — **Reversibility:** costly — undo restructures the whole layers panel in main.rs.
- **D-10:** Layer properties are edited inline on the row: eye (visibility), lock, colour swatch opening a colour picker, operation badge plus the hammer icon opening the operation popup. Row position doubles as order display. — **Reversibility:** reversible — control placement within the panel.
- **D-11:** Layers are reordered by drag handles on the rows; row position is the order. — **Reversibility:** reversible — interaction choice.
- **D-12:** Production order is shown as a "Pass N" badge on each layer that has an operation (its position among operation layers, per Phase 1 D-19); operation-less layers show no pass badge. — **Reversibility:** reversible — display only.
- **D-13:** Drag interactions are unified: dragging an operation from the palette onto a row assigns the operation; dragging a row by its handle reorders it. — **Reversibility:** reversible — drag routing in the panel.

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

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| UI-01 | Layer/Operations menu and add-layer flow that requires an operation | `Command::LayerAdd` must change to operation-less default (D-01); menu already has a Layers entry. See *Commands to Add/Change*. |
| UI-02 | Custom layer never silently defaults to `Cut`; chosen operation visible/editable | Replace `LayerAdd { name, kind: CutStyle::Cut }` with `LayerAdd { name, operation: Option<Operation> }`; add `LayerSetOperation`. See *Architecture Patterns*. |
| UI-03 | Add/remove one or many selected objects from a layer without moving geometry | New additive membership commands; geometry-invariance test strategy. See *Geometry Invariance*. |
| UI-04 | Reorder operation layers and see production order | `DocumentCommand::LayerMove` exists; Pass N derived from `production_sequence()`. See *Production Order*. |
| UI-05 | Edit layer colour, settings, visibility, lock from panel; reflected in canvas | `LayerSetColor`/`LayerSetVisible`/`LayerSetLocked` exist; `shape_visual` currently ignores `layer.color` — must be wired. |
| UI-06 | Object rows show memberships; "Unassigned" derived, not persisted | `Document::unassigned_shapes()` exists; panel builds it per frame. |
| UI-07 | Selecting a layer/object shows effective operation and its source in inspector | `craft_ui::patterns::Inspector` + `InspectorHeader`; compute from `Layer.operation` / memberships. |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Layer/operation model (members, operation, order) | Database / Storage (persisted document model in `craft-stencilcraft-document`) | — | Model owns invariants; UI must not persist derived state. |
| Membership assignment / operation set / reorder / property edits | API / Backend (`stencilcraft-engine` commands) | — | AGENTS.md "Everything is a command"; undo/validate live here. |
| Layers tree, palette, popup, Pass N, badges | Browser / Client (egui app + `craft-ui` widgets) | — | Display-only; dispatches commands after the UI pass. |
| Canvas colour/visibility reflection | Frontend Server / painter (`crates/ui-egui` `paint_document`) | Client (theme tokens) | Painting is a pure function of the document; layer colour must flow into `shape_visual`. |
| Unassigned derivation | Database / Storage (query) | Client (renders rows) | Derived from memberships, never stored. |
| Inspector effective operation | Client (panel) | API (query/read) | Reads document state; may add a query command if needed. |

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `craft-ui` (`craft_ui::list`, `patterns`, `composites`, `menu`) | git rev `8504e84805049d3af59baacf5239315eaaa74e50` | List/tree rows, toggles, drag-reorder, Inspector, panels | Already the app's design system; the pinned rev exposes `List`, `Reorder`, `TreeRow`, `Inspector`. [VERIFIED: worktrees/stencilcraft-planning/Cargo.toml:26] |
| `egui` / `eframe` | 0.37 (patched to `storytold/egui` rev `35a72e6b…`) | UI runtime, colour picker, drag payloads | Workspace pin. [VERIFIED: worktrees/stencilcraft-planning/Cargo.toml:22-23,114-127] |
| `stencilcraft-engine` | path crate | Command model + undo/redo | "Everything is a command" is a project non-negotiable. [VERIFIED: worktrees/stencilcraft-planning/crates/engine/Cargo.toml:2] |
| `stencilcraft-doc` → `craft-stencilcraft-document` | git rev (must move to a v2 rev) | v2 document model | Phase 1 landed the model in craft-libs; StencilCraft re-exports it. [VERIFIED: worktrees/stencilcraft-planning/crates/doc/src/lib.rs:11] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `kurbo` | 0.13 | Geometry (`BezPath`) | Geometry invariance comparisons. |
| `serde`/`serde_json` | 1 | Command/document wire format | New command variants must be serde-tagged. |
| `egui::color_picker` (`color_edit_button_srgba`) | egui 0.37 | Colour swatch picker | `craft-ui` has **no** colour-picker widget (verified: no `ColorPicker` in the crate). |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `craft_ui::list::List` (tree + reorder) | Raw `ListRow::show` loops (current panel) | Raw rows cannot do tree disclosure, selection rules, or drag-reorder without reimplementing them; `List` is the intended container. |
| New model membership helpers | Mutate `doc.layers[i].members` from engine | Model helpers are cleaner and upstreamable (AGENTS rule 9) but violate the stated "model untouched" boundary; see Open Question 2. |
| egui built-in colour picker | Add a craft-ui picker | Out of scope; no picker exists at the pinned rev. |

**Installation:** No new external packages are required. The only dependency change is the `craft-stencilcraft-document`/`-format` git rev (see Open Question 1).

**Version verification:** `rustc 1.98.0`, `cargo 1.98.0` (workspace requires 1.95). [VERIFIED: `rustc --version` / `cargo --version` this session]

## Package Legitimacy Audit

No new registry packages are introduced by this phase. The only dependency movement is a git `rev` bump for existing `storytold/craft-libs` crates (`craft-stencilcraft-document`, `craft-stencilcraft-format`), already present in the workspace.

| Package | Registry | Age | Downloads | Source Repo | Verdict | Disposition |
|---------|----------|-----|-----------|-------------|---------|-------------|
| (none — no new packages) | — | — | — | — | — | — |

**Packages removed due to [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** none

*Note:* the Phase 1 v2 rev is **uncommitted** in `refs/craft-libs` (`git status` shows modified `Cargo.toml`, `lib.rs`, `production.rs`, `canvas.rs`, READMEs, `Cargo.lock`). A `rev` bump requires first committing/publishing that work (PROJECT.md: the PR must reference a real upstreamable revision). [VERIFIED: git status in refs/craft-libs; `.planning/PROJECT.md` cross-repo constraint]

## Architecture Patterns

### System Architecture Diagram

```
              ┌──────────────────────────────┐
              │  Layers/Operations panel     │
              │  (apps/stencilcraft main.rs) │
              │                              │
  palette ───►│  List(tree): layer rows      │
  drag        │    ├ member shape rows       │
              │    └ "Unassigned" group      │
  popup  ────►│  eye / lock / colour / hammer│
              └──────────────┬───────────────┘
                             │ after UI pass: run_command(Command)
                             ▼
              ┌──────────────────────────────┐
              │  Engine::run(Command)        │
              │  - snapshot Document (undo)  │
              │  - validate, mutate doc      │
              │  - sync VectorCraft session  │
              └──────────────┬───────────────┘
                             │ mutates
                             ▼
              ┌──────────────────────────────┐
              │ craft-stencilcraft-document  │
              │  Document { shapes, layers } │
              │  Layer { operation, members, │
              │          visible, locked,    │
              │          color }             │
              └──────────────┬───────────────┘
                             │ read-only queries
              ┌──────────────┴───────────────┐
              ▼                              ▼
   unassigned_shapes()            production_sequence()
   (UI "Unassigned")              (Pass N + Phase 3 output)
                             │
                             ▼
              ┌──────────────────────────────┐
              │ ui-egui paint_document       │
              │  visible layers + layer      │
              │  colour + hidden shapes      │
              └──────────────────────────────┘
```

### Recommended Project Structure (files to touch)

```
worktrees/stencilcraft-planning/
├── crates/engine/src/lib.rs          # Command enum: LayerAdd/LayerSetVisible/LayerRemove; DocumentNew default
├── crates/engine/src/document.rs     # DocumentCommand: membership add/remove, set-operation, reorder
├── crates/ui-egui/src/lib.rs         # paint_document/shape_visual: read layer colour + members
└── apps/stencilcraft/src/
    ├── main.rs                       # layers_panel (tree), inspector_panel, command dispatch
    └── menus.rs                      # "Send to Layer…" submenu, Layers menu
refs/craft-libs/crates/stencilcraft-document/   # only if model helpers are added (Open Question 2)
```

### Pattern 1: Build a flat display order, dispatch after the UI pass

`craft_ui::list::List` takes `&[RowInfo<K>]` in display order and a closure that builds each `ListRow` by position/key. Build the tree as: each layer `RowInfo::group(Layer(i), open)` followed (when open) by its member shape rows `RowInfo::leaf(Shape(id)).at_depth(1)`, then an `Unassigned` group and its leaves. Collect `ListEvent`s and run commands after `show` returns (the existing panel already snapshots rows then dispatches; keep that).

```rust
// Source: craft_ui docs (crates/ui/src/list/view.rs:140-210) + main.rs:2573-2695 pattern
let rows: Vec<RowInfo<PanelKey>> = build_rows(&self.engine.doc, &self.open_groups);
let mut selection = std::mem::take(&mut self.layer_selection);
let out = List::new("stencilcraft-layers", "Layers", rows.as_slice())
    .tree(true)
    .selection(&mut selection)
    .reorder(Reorder::new(Axis::Vertical).accepts(|mv, _| accepts_reorder(mv)))
    .show(ui, |_, key| match key {
        PanelKey::Layer(i) => layer_row(&self.engine.doc, *i, ...),
        PanelKey::Shape(id) => shape_row(&self.engine.doc, *id, ...),
        PanelKey::Unassigned => ListRow::new(*key, "Unassigned"),
    });
self.layer_selection = selection;
for ev in out.events { /* Selection, Open, Row(Toggle), Reorder, ContextMenu, Activate */ }
```

[VERIFIED: craft-libs-1db3ee69db7fcb08/8504e84/crates/ui/src/list/view.rs:42-73 for `ListEvent` variants]

### Pattern 2: Row toggles and trailing controls

`ListRow` supports `ToggleColumn::eye(name, on)` / `ToggleColumn::lock(name, on)`, a `detail()` line (use it for operation + membership badges), `dimmed(!visible)`, `selected(bool)`, and `trailing_item(width, |ui| ...)` for the hammer icon and colour swatch. Toggle edits arrive as `ListEvent::Row(RowEvent::Toggle(EditEvent::Set { value, .. } | EditEvent::Commit { value, .. }))` with `ToggleEdit { column, row, on }`.

```rust
// Source: craft-libs-1db3ee69db7fcb08/8504e84/crates/ui/src/list/toggle.rs:194-202
pub struct ToggleEdit<K> {
    pub column: usize, // index in ListRow::toggle order
    pub row: K,
    pub on: bool,
}
```

[VERIFIED: toggle.rs:194-202; main.rs:2631-2660 shows the existing destructuring]

### Pattern 3: Drag routing (palette → row assign; handle → reorder)

`List::reorder` handles row drags and reports `ListEvent::Reorder(EditEvent<Move<K>>)`; `ListOutput::rows: Vec<(K, Rect)>` gives on-screen row rects for **foreign** payloads. Use a distinct payload type for the operation palette (e.g. `struct OpDrag(OperationKind);`) so it cannot be mistaken for `ListDrag`. On release, hit-test `out.rows` rects and dispatch `LayerSetOperation` for the row under the pointer. This satisfies D-13 without custom drag maths.

[VERIFIED: craft-libs-1db3ee69db7fcb08/8504e84/crates/ui/src/list/view.rs:88-135 (`ListOutput.rows`), behavior/reorder.rs `Move`/`ListDrag`]

### Pattern 4: Operation popup

Use `craft_ui::menu::Menu` (entries + `MenuResponse::chosen`) or `PopupList`. Open on `Add Layer` button when `response.clicked() && ui.input(|i| i.modifiers.shift)` (shortcut configurable), and from the hammer `trailing_item` for a specific layer. Choosing an entry dispatches `LayerSetOperation { index, operation: Some(preset) }`.

### Anti-Patterns to Avoid
- **Building rows from `s.style`/`layer.shapes`:** those fields do not exist in v2; derive from `members` and `operation`.
- **Persisting "Unassigned":** never create a layer for it; compute `doc.unassigned_shapes()` each frame.
- **Exclusive move on assign:** `ShapesMoveToLayer` drains shapes from other layers — it violates D-07. Do not reuse it.
- **Mutating the document in the UI pass:** dispatch commands after `show` so undo snapshots are clean.
- **Counting passes over all layers:** Pass N counts only `layer.produces()` layers (`Some(operation)` and not Guide).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Tree/list with selection, keyboard, AccessKit | Custom row loop | `craft_ui::list::List` + `RowInfo` + `TreeRow` | Handles focus, selection modes, disclosure, screen readers. |
| Drag-reorder | Manual pointer math | `List::reorder(Reorder::new(Axis::Vertical))` + `ListEvent::Reorder` | Drop markers, ghost, scrolling, Escape/cancel. |
| Eye/lock toggle cells | Custom buttons | `ToggleColumn::eye` / `::lock` | Sweep gestures, keys, AccessKit names. |
| Panel chrome | Custom headers | `Panel`/`PanelFooter`/`SectionHeader` | Theme tokens and layout already used. |
| Inspector | Custom property list | `craft_ui::patterns::Inspector` + `InspectorHeader` | Consistent header/empty state/scroll. |
| Colour parsing/format | `format!("#{:02x}…")` | `stencilcraft_doc::canvas::{parse_hex_color, format_hex_color}` | Validation + canonical form; already used by `layer.set-color`. [VERIFIED: worktrees/.../crates/engine/src/document.rs:15] |

## Runtime State Inventory

This is not a rename/refactor phase, but the cross-repo build state is runtime-relevant and must be planned explicitly.

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | Documents persisted as v2 by Phase 1; v1 files migrate in memory on load. No UI-owned stored state beyond existing panel settings. | None for Phase 2 beyond loading through the migrated model. |
| Live service config | None — desktop app; no server/DB. Control/MCP surfaces are local. | None. |
| OS-registered state | None. | None. |
| Secrets/env vars | None introduced. | None. |
| Build artifacts / installed packages | Worktree pins v1 (`75c70fa6…`); Phase 1 v2 is uncommitted in `refs/craft-libs`; `Cargo.lock` records v1. | Commit/publish Phase 1, bump the pin (or add a local `[patch]`), regenerate `Cargo.lock`. **Blocking prerequisite.** |

**Nothing found in a category:** Stored data, live service config, OS-registered state, secrets — verified by inspecting the workspace layout and Phase 1 artifacts.

## Common Pitfalls

### Pitfall 1: Assuming the worktree already builds on Phase 1
**What goes wrong:** Planning UI tasks that call `doc.unassigned_shapes()` / `Layer.members` against a worktree pinned to v1 — nothing compiles.
**Why it happens:** Phase 1 changed only `refs/craft-libs`; the worktree pin was deliberately left for Phase 2 ("Phase 2 dev setup").
**How to avoid:** Wave 0 must commit/branch Phase 1 in craft-libs, bump the worktree pin (or `[patch]`), and migrate v1 call sites.
**Warning signs:** `no field 'shapes' on type 'Layer'`, `no field 'style' on type 'Shape'`, `no field 'kind'`.

### Pitfall 2: Reusing `ShapesMoveToLayer` for assignment
**What goes wrong:** It drains the shape from every other layer (`l.shapes.drain(..).partition(...)`) — exclusive move, breaking engrave-then-cut.
**How to avoid:** Add additive membership commands; keep other memberships.
**Warning signs:** a shape disappears from a second layer after assignment.

### Pitfall 3: Implicit Cut on new documents/layers
**What goes wrong:** `DocumentNew` still creates `Layer::new("cut", CutStyle::Cut)` and `LayerAdd` still hardcodes `CutStyle::Cut`.
**How to avoid:** Change both to `Layer::new("base", None)` / `operation: None`; add `LayerSetOperation` for the explicit choice.
**Warning signs:** a new layer shows "Cut" before the user chose anything.

### Pitfall 4: Geometry mutation during assignment
**What goes wrong:** A command rebuilds/replaces `Shape` objects (or writes `style` like the v1 path) while assigning membership.
**How to avoid:** Only mutate `Layer.members`; never touch `Document.shapes`. Test by comparing shape paths before/after.
**Warning signs:** bounds or path equality fails after assign/remove.

### Pitfall 5: Pass N computed over all layers
**What goes wrong:** Operation-less layers shift the numbers.
**How to avoid:** Enumerate only `doc.layers.iter().filter(|l| l.produces())`; Guide and `None` layers get no badge.

### Pitfall 6: Undo path skipped for new commands
**What goes wrong:** New top-level `Command` variants mutate `doc` without a history snapshot or session resync.
**How to avoid:** Implement layer/membership mutations as `DocumentCommand` variants (document.rs captures history, validates, and `sync_session_from_doc()`s) or add them to the `rebuild_session` match in `Engine::run`.

### Pitfall 7: Cross-repo path validation
**What goes wrong:** Plans say "run `git ls-files`" or use repo-relative paths; `rustfactor` is a separate git repo from the worktree, so those paths do not resolve.
**How to avoid:** Express worktree paths absolutely or via `--manifest-path /home/r2d2/Sandbox/lasercraft/worktrees/stencilcraft-planning/Cargo.toml`; do not validate worktree files with `rustfactor` git.

## Code Examples

### Operation-less layer add (D-01 / D-18)
```rust
// Source: Phase 1 model — refs/craft-libs/crates/stencilcraft-document/src/lib.rs:117-119
pub fn new(name: impl Into<String>, operation: Option<Operation>) -> Self {
    Self { name: name.into(), operation, preset_id: None, preset_linked: false,
           visible: true, locked: false, color: None, members: Vec::new() }
}
// engine: Layer::new(name, None);  DocumentNew default: Layer::new("base", None)
```

### Derived Unassigned (UI-06)
```rust
// Source: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:489-493
pub fn unassigned_shapes(&self) -> impl Iterator<Item = &Shape> {
    let assigned: BTreeSet<u64> =
        self.layers.iter().filter(|l| l.operation.is_some())
            .flat_map(|l| l.members.iter().copied()).collect();
    self.shapes.iter().filter(move |s| !assigned.contains(&s.id))
}
```

### Production sequence / Pass N (UI-04 / D-12)
```rust
// Source: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:499-504
pub fn production_sequence(&self) -> impl Iterator<Item = (&Layer, Vec<&Shape>)> {
    self.layers.iter().filter(|l| l.produces()).map(|layer| {
        let members = layer.members.iter().filter_map(|id| self.get_shape(*id)).collect();
        (layer, members)
    })
}
// Pass N = 1-based index into production_sequence().
```

### Additive membership command (new)
```rust
// Proposed DocumentCommand (engine/src/document.rs)
#[serde(rename = "layer.add-members")]
LayerAddMembers { layer: usize, ids: Vec<u64> },
#[serde(rename = "layer.remove-members")]
LayerRemoveMembers { layer: usize, ids: Vec<u64> },
#[serde(rename = "layer.set-operation")]
LayerSetOperation { index: usize, operation: Option<Operation> },
// Apply: validate index/ids, reject locked target if required, push ids not already present
// (append preserves intra-layer order D-20), remove by retain. Never touch Document.shapes.
```

### Canvas reads layer colour/visibility (UI-05)
```rust
// Current v1 paint loop — worktrees/stencilcraft-planning/crates/ui-egui/src/lib.rs:622-627
for layer in &doc.layers {
    if !layer.visible { continue; }
    for shape in layer.shapes.iter().filter(|shape| !shape.hidden) { ... }
}
// v2: iterate doc.layers; resolve members -> doc.get_shape(id); pass layer.color
// into shape_visual (currently matches shape.style at lib.rs:544-547).
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `Layer.shapes: Vec<Shape>`, `Shape.style` | Geometry once in `Document.shapes`; `Layer.members: Vec<u64>`; `Layer.operation: Option<Operation>` | Phase 1 (craft-libs 0.2.0, uncommitted) | All v1 call sites in the worktree break; UI must read memberships. |
| Exclusive "move to layer" | Additive N:M membership | Phase 1 D-02/D-07 | Assignment must not remove other memberships. |
| Implicit Cut layer | Operation-less base layer | Phase 1 D-18 | `DocumentNew`/`LayerAdd` must change. |
| `template` flag | `Operation::Guide` (non-output) | Phase 1 D-17 | `LayerSetTemplate` is legacy; do not build UI on it. |

**Deprecated/outdated:** `CutStyle`-based `ShapesSetStyle`, `LayerSetTemplate`, `Shape.style`, `Layer.kind`, `Layer.shapes` — all v1-only. `ShapeProduction.operation` was removed in Phase 1.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Phase 2 owns the full worktree v1→v2 migration (~178 non-test call sites across engine/ui-egui/layout/apps/gcode/mcp/studio/dxf). | Summary, Runtime State | Scope explosion or an unbuildable phase; needs user confirmation. |
| A2 | Membership mutation helpers live in the engine (model untouched), not in craft-libs. | Don't Hand-Roll, Open Q2 | If model helpers are required, Phase 2 must edit craft-libs and re-publish. |
| A3 | A minimal built-in operation-preset catalogue is acceptable (no profiles system exists in the worktree yet). | Pattern 4, Open Q3 | D-07 says presets come from the profiles system; if that must ship first, Phase 2 grows. |
| A4 | Changing `layer.add` serde shape is acceptable (vs. adding a new command to preserve old MCP/control clients). | Commands | External control/MCP clients could break. |
| A5 | egui's built-in colour picker is acceptable for the swatch. | Standard Stack | A craft-ui picker may be expected; none exists at the pinned rev. |
| A6 | The Phase 1 rev can be committed to a branch that the worktree can pin. | Runtime State | If not upstreamable, a local `[patch]` path is needed (PROJECT.md allows dev patches). |

## Open Questions (RESOLVED)

1. **Pin strategy / Phase 1 publication** — RESOLVED (D-14): commit Phase 1 v2 to a `phase-1-v2-model` branch in `refs/craft-libs`; worktree pins that sha via `file:///home/r2d2/Sandbox/lasercraft/refs/craft-libs` (branch not pushed; switch URL to GitHub after push).
2. **Where membership helpers live** — RESOLVED (D-16): engine-local (`crates/engine`), mutating public v2 `Document.layers`/`shapes`; model API untouched.
3. **Operation preset catalogue** — RESOLVED (D-17): minimal built-in catalogue (Cut, Engrave, Score, Guide, …) from `OperationSettings::default()`, `preset_id` unset; no profiles crate.
4. **Scope split** — RESOLVED (D-15): fold the v1→v2 migration into Phase 2 Wave 0 (`02-01-PLAN.md`) with per-crate tasks and a compile gate; no separate prerequisite phase.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Rust toolchain | Build/test | ✓ | rustc 1.98.0 / cargo 1.98.0 | — |
| Git worktree | Source access | ✓ | worktree of `refs/stencilcraft` on `planning` | — |
| `craft-libs` local clone | Phase 1 model | ✓ | HEAD `249a41b` + uncommitted v2 | `[patch]` path if unpushed |
| Display (egui) | Manual UI run | ✗ (headless) | — | Unit-test pure paint/row helpers; `cargo run -p stencilcraft` on a GUI host |

**Missing dependencies with no fallback:** none blocking code/test (GUI visual check needs a display but is not required for automated validation).

**Missing dependencies with fallback:** GUI display → headless unit tests + CLI/control paths.

## Validation Architecture

> `workflow.nyquist_validation` is `true` in `.planning/config.json`; this section is required.

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Rust built-in test harness (`cargo test`) |
| Config file | none — `xtask/src/main.rs` defines the gates |
| Quick run command | `cargo test -p stencilcraft-engine --lib --locked` (worktree) |
| Full suite command | `cargo xtask ci` from the worktree root (fmt + clippy -D warnings + workspace tests + layers) |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| UI-01 | Add layer creates `operation: None`; menu entry exists | unit | `cargo test -p stencilcraft-engine layer_add_is_operationless --locked` | ❌ Wave 0 |
| UI-02 | `LayerSetOperation` sets/clears; no implicit Cut on `DocumentNew` | unit | `cargo test -p stencilcraft-engine document_new_has_no_cut --locked` | ❌ Wave 0 |
| UI-03 | Add/remove members is additive and preserves geometry | unit | `cargo test -p stencilcraft-engine assign_preserves_geometry --locked` | ❌ Wave 0 |
| UI-04 | `LayerMove` changes `production_sequence()` order; Pass N derived | unit | `cargo test -p stencilcraft-engine reorder_changes_pass_order --locked` | ❌ Wave 0 |
| UI-05 | Colour/visible/lock commands round-trip; paint uses layer colour | unit | `cargo test -p stencilcraft-engine layer_properties_roundtrip --locked` + `cargo test -p stencilcraft-ui-egui layer_colour_affects_visual --locked` | ❌ Wave 0 |
| UI-06 | `unassigned_shapes()` excludes any operation-layer member; UI rows derived | unit | `cargo test -p craft-stencilcraft-document unassigned --locked` (refs/craft-libs) + `cargo test -p stencilcraft-ui-egui unassigned_rows --locked` | ✅ model test exists; ❌ UI helper |
| UI-07 | Effective operation + source computed from layer/memberships | unit | `cargo test -p stencilcraft-ui-egui effective_operation --locked` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `cargo test -p stencilcraft-engine --lib --locked`
- **Per wave merge:** `cargo test -p stencilcraft-engine -p stencilcraft-ui-egui --locked`
- **Phase gate:** `cargo xtask ci` green before `/gsd-verify-work`; plus `cargo test -p craft-stencilcraft-format -p craft-stencilcraft-document --locked` in `refs/craft-libs` if the model is touched.

### Wave 0 Gaps
- [ ] Worktree re-pin + v1→v2 call-site migration (compile gate).
- [ ] `crates/engine` tests for membership/operation/order commands (geometry invariance, additive semantics).
- [ ] `crates/ui-egui` pure helpers extracted for testing: `build_layer_rows(doc)`, `pass_number(doc, index)`, `effective_operation(doc, selection)`, layer-colour-aware `shape_visual`.
- [ ] (If model helpers added) `craft-stencilcraft-document` membership tests in `tests/it/main.rs`.

## Security Domain

`security_enforcement: true`, `security_asvs_level: 1`.

### Applicable ASVS Categories
| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | Desktop app; no auth surface. |
| V3 Session Management | no | No sessions. |
| V4 Access Control | no | Local app; control/MCP are local, out of this phase's scope. |
| V5 Input Validation | yes | Validate layer indices with `.get`, shape ids exist, `Operation::validate()`, `parse_hex_color` for `#rrggbb`; reject locked-layer edits; treat drag payloads as untrusted (no panic). |
| V6 Cryptography | no | None. |

### Known Threat Patterns for Rust/egui + document model
| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Malicious `.stencilcraft` on load (huge/duplicate ids, NaN/∞, dangling members) | Tampering / DoS | `load_json` → `validate_model` returns calm `DocError`; Phase 1 hostile tests cover this. Never `unwrap` in shipped code. |
| Control/MCP constructing new commands with bad params | Tampering | Serde-tagged commands + engine `EngineError::Invalid`; bounds-check indices/ids. |
| Colour string injection | Tampering | `parse_hex_color` + `format_hex_color` canonicalisation. |
| Drag/drop payload confusion | Spoofing | Distinct payload types; hit-test only known row rects; no filesystem paths from drops. |
| Locked layer silently edited via assignment | Elevation of privilege (local) | Reject membership changes to locked layers (mirror existing `shapes.move-to-layer` guard). |

## Sources

### Primary (HIGH confidence)
- `refs/craft-libs/crates/stencilcraft-document/src/lib.rs:90-136,382-411,486-504` — v2 `Layer`, `Document`, `unassigned_shapes`, `production_sequence` (read this session; Phase 1 working tree).
- `refs/craft-libs/crates/stencilcraft-document/src/production.rs:85-169` — `Operation` variants + accessors.
- `worktrees/stencilcraft-planning/crates/engine/src/lib.rs:230-240,1359-1378` — `LayerAdd`/`LayerSetVisible`/`LayerRemove`, `DocumentNew` default.
- `worktrees/stencilcraft-planning/crates/engine/src/document.rs:68-95,693-770` — layer commands and their implementations.
- `worktrees/stencilcraft-planning/apps/stencilcraft/src/main.rs:2573-2695` — current flat layers panel.
- `worktrees/stencilcraft-planning/apps/stencilcraft/src/menus.rs:231-237` — `todo("Send to Layer…")`.
- `worktrees/stencilcraft-planning/crates/ui-egui/src/lib.rs:533-550,621-627` — `shape_visual` + paint loop (v1).
- `worktrees/stencilcraft-planning/Cargo.toml:22-45` — egui/craft-ui/craft-stencilcraft rev pins; `:114-127` — egui patch.
- `craft-libs-1db3ee69db7fcb08/8504e84/crates/ui/src/list/{view.rs:42-73, toggle.rs:194-202, row.rs, tree.rs}` — `List`/`ListEvent`/`ToggleEdit`/`TreeRow`.
- `craft-libs-1db3ee69db7fcb08/8504e84/crates/ui/src/patterns/inspector.rs` — `Inspector`/`InspectorHeader`.
- `worktrees/stencilcraft-planning/AGENTS.md` — no-panic, everything-is-a-command, layering, `cargo xtask ci`.
- `refs/craft-libs/AGENTS.md` — craft-libs rules (reuse, upstreamable revs, tests).

### Secondary (MEDIUM confidence)
- `.planning/phases/01-operation-first-document-model/01-RESEARCH.md` — Q1 answer: "StencilCraft `[patch]`/branch wiring is Phase 2 dev setup".
- `.planning/PROJECT.md` — cross-repo `[patch]`/upstreamable revision constraint.
- `.planning/phases/01-operation-first-document-model/01-0*-SUMMARY.md` — Phase 1 model changes and verify command.

### Tertiary (LOW confidence)
- Operation-preset catalogue shape (A3) — no profiles system found in the worktree; recommendation is a built-in minimal catalogue.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — pinned revs and widget APIs read directly.
- Architecture: HIGH for the model/command split; MEDIUM for the exact UI widget composition (discretion area).
- Pitfalls: HIGH — the v1/v2 field mismatch and exclusive-move behaviour are verified in source.
- Build/integration scope: MEDIUM — the migration is real, but its ownership (Phase 2 vs prerequisite) is an open decision.

**Research date:** 2026-11-24
**Valid until:** ~2026-12-08 (fast-moving cross-repo pins; re-check if craft-libs moves).

## RESEARCH COMPLETE

**Highest-risk unknowns:**
1. **Build boundary:** the worktree pins v1 while Phase 1's v2 model is uncommitted — the re-pin + ~178-site migration must be scoped before any UI work.
2. **Membership command home:** engine-local mutation vs. upstream `Document` helpers (phase boundary vs. craft-libs reuse rule).
3. **Preset catalogue:** D-07 says profiles, but no profiles system exists — Phase 2 must ship a built-in catalogue or depend on one.
4. **Undo/session resync:** new membership commands must go through the `DocumentCommand` path (or the `rebuild_session` list) or undo will desync the VectorCraft session.
5. **Cross-repo path expression:** plans must use absolute worktree paths/`--manifest-path`; `rustfactor` git cannot validate worktree files.
