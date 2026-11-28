# Phase 2: Layer/Operations UI and Assignment - Pattern Map

**Mapped:** 2026-11-24
**Files analyzed:** 13 primary targets (+ ~25 Wave 0 migration files)
**Analogs found:** 11 / 13 primary targets

**Source roots used for citation**

| Root | What it is |
|------|------------|
| `/home/r2d2/Sandbox/lasercraft/worktrees/stencilcraft-planning/` | The real tracked source tree (v1 worktree). All app/engine/UI citations below are relative to this root unless absolute. |
| `/home/r2d2/Sandbox/lasercraft/refs/craft-libs/` | Tracked clone whose working tree holds the uncommitted Phase 1 v2 model (`crates/stencilcraft-document`, `crates/stencilcraft-format`). |
| `/home/r2d2/.cargo/git/checkouts/craft-libs-1db3ee69db7fcb08/8504e84/` | Resolved Cargo checkout for the pinned `craft-ui` rev `8504e84…` (`worktrees/.../Cargo.toml:26`). This is dependency source, not a repo mirror; it is the only tree containing `craft_ui::list`, `TreeRow`, `Inspector` and `menu::Menu` at the pinned rev. |

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|---|---|---|---|---|
| `crates/engine/src/document.rs` | service / command handler | CRUD (document mutation) | same file: layer commands + `run` undo wrapper | exact |
| `crates/engine/src/lib.rs` | controller / command dispatcher | request-response | same file: `LayerAdd`/`LayerSetVisible`/`LayerRemove` handlers | exact |
| `crates/engine/src/production.rs` | service | CRUD (migration only) | v2 `Operation` model; existing `ProductionCommand` | role-match |
| `crates/ui-egui/src/lib.rs` | renderer / painter | transform | same file: `shape_visual` + `paint_document` loop | exact |
| `crates/ui-egui/src/layers.rs` *(new, optional)* | utility (pure helpers) | transform | `crates/ui-egui/src/lib.rs` pure helpers + tests | role-match |
| `apps/stencilcraft/src/main.rs` | component + controller | request-response | same file: `layers_panel`, `inspector_panel`, `run_command` | exact |
| `apps/stencilcraft/src/operations.rs` *(new)* | config / catalogue | transform | `ActiveTool::{icon,label}` + `MAT_CHOICES` | exact |
| `apps/stencilcraft/src/menus.rs` | config / menu definition | request-response | same file: Object menu + `item`/`when`/`sub`/`todo` helpers | exact |
| `apps/stencilcraft/src/panels.rs` | controller / action dispatcher | request-response | same file: `EditAction` + `handle_edit_action` | exact |
| `Cargo.toml`, `Cargo.lock` | config | n/a | same file: craft-libs rev pins | exact |
| Wave 0 migration set (~25 `.rs` files) | mixed | mixed | v2 model in `refs/craft-libs` + current v1 call sites | role-match |
| `crates/engine/src/document.rs` `mod tests` | test | CRUD | same file: `three()` / `run()` layer tests | exact |
| `crates/engine/tests/production.rs` | test | CRUD | same file: JSON envelope + geometry assertions | exact |
| `refs/craft-libs/.../tests/it/main.rs` *(only if model helpers added)* | test | model | existing Phase 1 model tests | exact |

---

## Pattern Assignments

### 1. `crates/engine/src/document.rs` (service, CRUD)

**Analog:** the same file's existing layer commands.

**Command enum pattern** (lines 43-95) — add the new variants next to the existing layer block. Serde tag convention is `#[serde(tag = "command", content = "params")]` + kebab/dot renames:

```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(tag = "command", content = "params")]
pub enum DocumentCommand {
    // ...
    /// Move shapes to the top of layer `layer`; they take that layer's operation.
    #[serde(rename = "shapes.move-to-layer")]
    ShapesMoveToLayer { ids: Vec<u64>, layer: usize },
    /// Move layer `from` to position `to` (both indices into the current list).
    #[serde(rename = "layer.move")]
    LayerMove { from: usize, to: usize },
    #[serde(rename = "layer.rename")]
    LayerRename { index: usize, name: String },
    /// Lock or unlock a layer: its shapes cannot be selected or edited while locked.
    #[serde(rename = "layer.set-locked")]
    LayerSetLocked { index: usize, locked: bool },
    /// Make a layer a template (guide) layer: shown, never exported or cut.
    #[serde(rename = "layer.set-template")]
    LayerSetTemplate { index: usize, template: bool },
    /// Set or clear (`null`) a layer's material colour, `#rrggbb`.
    #[serde(rename = "layer.set-color")]
    LayerSetColor { index: usize, color: Option<String> },
```

Add (naming follows the same dot convention; **additive**, never a move):

```rust
#[serde(rename = "layer.add-members")]
LayerAddMembers { layer: usize, ids: Vec<u64> },
#[serde(rename = "layer.remove-members")]
LayerRemoveMembers { layer: usize, ids: Vec<u64> },
#[serde(rename = "layer.set-operation")]
LayerSetOperation { index: usize, operation: Option<Operation> },
```

**Undo / validation wrapper** (lines 474-509) — do not bypass it. Every mutating `DocumentCommand` gets one snapshot, validation, session resync and rollback for free:

```rust
pub(crate) fn run(engine: &mut Engine, command: DocumentCommand) -> Result<Effect, EngineError> {
    if let Some(result) = query(engine, &command) { return result; }
    // ...
    let before = engine.capture_history_entry(false, true);
    let state = engine.document_state.clone();
    let result = apply(engine, command).and_then(|effect| {
        engine.doc.groups = engine.doc.pruned_groups();
        engine.doc.validate().map_err(|e| invalid(e.to_string()))?;
        Ok(effect)
    });
    match result {
        Ok(effect) => {
            engine.sync_session_from_doc();
            engine.history.rebase_to_snapshots();
            engine.history.push_snapshot(HistoryEntry {
                vectorcraft: false, rebuild_session: true, ..before
            });
            Ok(effect)
        }
        Err(error) => {
            engine.doc = before.snapshot;
            engine.restore_mappings(before.mappings);
            engine.document_state = state;
            Err(error)
        }
    }
}
```

**Apply-arm pattern** (lines 719-762) — validate before mutation, use `.get`/`.get_mut`, return a small `Effect`:

```rust
DocumentCommand::LayerMove { from, to } => {
    let from = layer_index(engine, from, "layer.move")?;
    let to = layer_index(engine, to, "layer.move")?;
    let layer = engine.doc.layers.remove(from);
    engine.doc.layers.insert(to, layer);
    changed
}
DocumentCommand::LayerSetLocked { index, locked } => {
    let index = layer_index(engine, index, "layer.set-locked")?;
    if let Some(l) = engine.doc.layers.get_mut(index) { l.locked = locked; }
    changed
}
DocumentCommand::LayerSetColor { index, color } => {
    let index = layer_index(engine, index, "layer.set-color")?;
    let color = match color {
        None => None,
        Some(text) => Some(format_hex_color(parse_hex_color(&text).ok_or_else(
            || invalid("layer.set-color: color must be #rrggbb".to_string()),
        )?)),
    };
    if let Some(l) = engine.doc.layers.get_mut(index) { l.color = color; }
    changed
}
```

**Validation helpers** (lines 624-648):

```rust
fn layer_index(engine: &Engine, index: usize, command: &str) -> Result<usize, EngineError> {
    if index < engine.doc.layers.len() { Ok(index) }
    else { Err(invalid(format!("{command}: no layer {index} (document has {})", engine.doc.layers.len()))) }
}

/// Unique, existing, unlocked shape ids.
fn editable_ids(engine: &Engine, ids: &[u64], command: &str) -> Result<Vec<u64>, EngineError> { /* unique_ids + get_shape + shape_locked */ }
```

**Anti-pattern — do not copy** (lines 693-718): `ShapesMoveToLayer` **drains** the shape from every layer, which violates D-07. New membership arms only push ids into the target's `members` and `retain` on remove; they never touch `Document.shapes` and never remove from other layers.

```rust
// v1 exclusive move: D-07 forbids reusing this for assignment.
for l in &mut doc.layers {
    let (take, keep): (Vec<_>, Vec<_>) = l.shapes.drain(..).partition(|s| selected.contains(&s.id));
    l.shapes = keep;
    moved.extend(take);
}
```

**Target model excerpt** (`/home/r2d2/Sandbox/lasercraft/refs/craft-libs/crates/stencilcraft-document/src/lib.rs:95-136`):

```rust
pub struct Layer {
    pub name: String,
    pub operation: Option<Operation>,
    // preset_id / preset_linked ...
    pub visible: bool,
    pub locked: bool,
    pub color: Option<String>,
    /// Member shape ids, in intra-layer order (D-20). Geometry lives once in
    /// `Document::shapes`; a shape may be a member of many layers (D-02).
    pub members: Vec<u64>,
}
pub fn produces(&self) -> bool {
    self.operation.as_ref().is_some_and(|op| !op.is_guide())
}
```

**Effect pattern:** follow `Effect::ShapesStyled { ids }` / `Effect::LayerRemoved { index }` (`crates/engine/src/lib.rs:527-560`); add e.g. `LayerMembersChanged { layer: usize, ids: Vec<u64> }`, `LayerOperationChanged { index: usize }`.

---

### 2. `crates/engine/src/lib.rs` (controller, request-response)

**Analog:** same file, top-level layer handlers.

**Command enum** (lines 232-249) — current v1 payload:

```rust
/// Add a layer with a name and a production kind.
#[serde(rename = "layer.add")]
LayerAdd { name: String, kind: CutStyle },
/// Set visibility of a layer by index.
#[serde(rename = "layer.set-visible")]
LayerSetVisible { index: usize, visible: bool },
/// Remove a layer by index (at least one layer must remain).
#[serde(rename = "layer.remove")]
LayerRemove { index: usize },
// ...
#[serde(rename = "document.new")]
DocumentNew { width_mm: f64, height_mm: f64 },
```

Phase 2: change `LayerAdd` to an operation-less add (D-01) — recommended shape `LayerAdd { name: String }` or `LayerAdd { name: String, operation: Option<Operation> }`; the popup/assign flow uses `DocumentCommand::LayerSetOperation`.

**Handler pattern** (lines 1297-1328) — validate, mutate, return an `Effect`; never `unwrap`:

```rust
Command::LayerAdd { name, kind } => {
    if name.trim().is_empty() {
        return Err(EngineError::Invalid("layer.add: name cannot be empty".to_string()));
    }
    self.add_vector_layer(&name, kind, "layer.add")?;
    Ok(Effect::LayerAdded)
}
Command::LayerSetVisible { index, visible } => {
    let Some(layer) = self.doc.layers.get_mut(index) else {
        return Err(EngineError::Invalid(format!("layer.set-visible: no layer at index {index}")));
    };
    layer.visible = visible;
    Ok(Effect::LayerVisibilityChanged { index, visible })
}
Command::LayerRemove { index } => {
    if index >= self.doc.layers.len() { /* Invalid */ }
    if self.doc.layers.len() <= 1 { /* Invalid: keep at least one */ }
    self.doc.layers.remove(index);
    Ok(Effect::LayerRemoved { index })
}
```

`add_vector_layer` (lines 2281-2318) performs the VectorCraft `layer.new` call then inserts `Layer::new(name, kind)`. Under v2 it must take `Option<Operation>` (or no operation) and insert `Layer::new(name, operation)`.

**Implicit-Cut fix** (`DocumentNew`, lines 1359-1383):

```rust
// v1
layers: vec![stencilcraft_doc::Layer::new("cut", CutStyle::Cut)],
// v2 (matches model default at refs/.../lib.rs:451-453)
layers: vec![stencilcraft_doc::Layer::new("base", None)],
```

**Session rebuild list** (lines 734-759) — only for commands that bypass `DocumentCommand::run`. If a new layer/membership command is implemented at top level instead of in `document.rs`, it MUST be added to this `rebuild_session` match, or the VectorCraft session and undo desync (RESEARCH Pitfall 6). Preferred: put it in `DocumentCommand`.

**Serde convention:** every variant carries `#[serde(rename = "dot.name")]`; params must be plain serialisable data so MCP/control can construct them (AGENTS "Everything is a command").

**Legacy to remove/supersede:** `ShapesSetStyle` (line 146) and its handler (lines 1041-1096) implement the v1 exclusive `shape.style`/`layer.kind` model. It cannot survive v2; menu `SetStyle` (see §8) must be replaced by layer-operation assignment.

---

### 3. `crates/engine/src/production.rs` (service, migration)

**Analog:** the v2 model's `Operation` + existing `ProductionCommand` enum.

Current file already imports `Operation` from `stencilcraft_doc` but treats it as a **per-shape** field (`s.production.operation = Some(Operation::PrintThenCut)` at line 237; `StickerCut → Operation::KissCut/DieCut` at lines 358-359). In v2, `ShapeProduction.operation` is gone; the equivalent is membership in a layer whose `operation` is `Some(KissCut/DieCut/…)`.

Pattern to copy for the migration: keep `ProductionCommand`/`ProductionEffect` dispatch as-is (`crates/engine/src/production.rs:1-60`), but every place that writes `shape.production.operation` must instead resolve/create an operation layer and insert the shape id into `Layer.members` (additive). Do not delete shapes or move geometry.

---

### 4. `crates/ui-egui/src/lib.rs` (renderer, transform)

**Analog:** same file's `shape_visual` + paint loop.

**Current v1 visual** (lines 532-550):

```rust
fn shape_visual(
    shape: &stencilcraft_doc::Shape,
    view: &View,
    kerf_preview: bool,
    palette: Palette,
) -> (egui::Color32, f32) {
    let line_color = if kerf_preview {
        palette.kerf_line
    } else {
        match shape.style {
            stencilcraft_doc::CutStyle::Cut => palette.cut_line,
            stencilcraft_doc::CutStyle::Engrave => palette.engrave_line,
        }
    };
    (line_color, 1.5_f32.max(0.4 * view.zoom))
}
```

**Current paint loop** (lines 621-650):

```rust
for layer in &doc.layers {
    if !layer.visible { continue; }
    for shape in layer.shapes.iter().filter(|shape| !shape.hidden) {
        let (mut color, mut width) = shape_visual(shape, view, options.kerf_preview, palette);
        if selection.selected_shapes.contains(&shape.id) {
            color = palette.selection;
            width = width.max(2.5);
        }
        // path_to_polylines -> screen ...
    }
}
```

**v2 pattern:** iterate `doc.layers.iter().filter(|l| l.visible)`, resolve each `layer.members` id through `doc.get_shape(id)`, skip `shape.hidden`, and pass the layer's `Operation`/`color` into `shape_visual` (UI-05). Shape visual is a pure function of (shape, layer, view, options, palette) — keep it testable without a display.

**Palette tokens** (lines 87-101): `cut_line`, `engrave_line`, `kerf_line`, `selection`. A layer colour (`#rrggbb`) should override the operation token only when set; parse with `stencilcraft_doc::canvas::parse_hex_color` (already used by the engine's `layer.set-color`).

**Test analog** (lines 1539-1547): pure helper tests need no egui window:

```rust
let (plain_col, _) = shape_visual(shape, &view, false, palette);
let (kerf_col, _) = shape_visual(shape, &view, true, palette);
assert_ne!(plain_col, kerf_col);
```

Add equivalents: `layer_colour_affects_visual`, `unassigned_rows`, `effective_operation`.

---

### 5. `crates/ui-egui/src/layers.rs` *(new, optional pure-helper module)*

**Analog:** `crates/ui-egui/src/lib.rs` pure helpers + `mod tests` (line 1296). There is no existing layers module; if the planner prefers, these helpers can live at the bottom of `lib.rs`.

Recommended pure functions (RESEARCH Validation Architecture / Wave 0 gaps):

```rust
pub fn pass_number(doc: &Document, layer_index: usize) -> Option<usize>;
pub fn build_layer_rows(doc: &Document, open: &BTreeSet<usize>) -> Vec<LayerRow>;
pub fn effective_operation(doc: &Document, selection: &[u64]) -> Option<(u64, Operation)>;
```

Derive order only from the model; never store:

```rust
// refs/craft-libs/crates/stencilcraft-document/src/lib.rs:489-504
pub fn unassigned_shapes(&self) -> impl Iterator<Item = &Shape> {
    let assigned: BTreeSet<u64> = self.layers.iter()
        .filter(|l| l.operation.is_some())
        .flat_map(|l| l.members.iter().copied()).collect();
    self.shapes.iter().filter(move |s| !assigned.contains(&s.id))
}
pub fn production_sequence(&self) -> impl Iterator<Item = (&Layer, Vec<&Shape>)> {
    self.layers.iter().filter(|l| l.produces()).map(|layer| {
        let members = layer.members.iter().filter_map(|id| self.get_shape(*id)).collect();
        (layer, members)
    })
}
```

Pass N = 1-based index into `production_sequence()`; `None` for `!layer.produces()` (D-12, RESEARCH Pitfall 5).

---

### 6. `apps/stencilcraft/src/main.rs` (component + controller, request-response)

**Analog:** same file's current flat panel.

**Snapshot → draw → dispatch-after pattern** (lines 2573-2698). Build plain data first, draw rows, collect toggles/clicks, run commands only after the UI pass:

```rust
pub fn layers_panel(&mut self, ui: &mut egui::Ui) {
    let mut add_layer = false;
    let mut delete_layer = false;
    let mut selected_idx = self.selected_layer;
    let mut toggles = Vec::new();

    let can_delete = self.engine.doc.layers.len() > 1;
    let layer_data: Vec<(usize, String, bool)> = self.engine.doc.layers.iter()
        .enumerate().map(|(i, l)| (i, l.name.clone(), l.visible)).collect();
    // ...
    Panel::new("stencilcraft-layers-panel")
        .footer_label("Layer actions")
        .footer(|ui| { PanelFooter::show(ui, |ui| {
            if ui.add(IconButton::new(Icon::Plus, "Add Layer")).clicked() { add_layer = true; }
            if ui.add(IconButton::new(Icon::Trash, "Delete Layer").enabled(can_delete)).clicked() { delete_layer = true; }
        }); })
        .show(ui, |ui| { /* ListRow rows + SectionHeader("Objects") */ });

    // Commands AFTER the UI pass
    if add_layer {
        self.run_command(Command::LayerAdd {
            name: format!("Layer {}", self.engine.doc.layers.len() + 1),
            kind: CutStyle::Cut,          // Phase 2: remove; operation-less add (D-01)
        });
    }
    if delete_layer && let Some(idx) = self.selected_layer {
        self.run_command(Command::LayerRemove { index: idx });
    }
    for (idx, vis) in toggles {
        self.run_command(Command::LayerSetVisible { index: idx, visible: vis });
    }
}
```

**Toggle destructuring** (lines 2630-2637 / 2651-2658) — reuse this for the tree's eye/lock cells:

```rust
for ev in row.events {
    if let craft_ui::list::RowEvent::Toggle(
        EditEvent::Commit { value, .. } | EditEvent::Set { value, .. },
    ) = ev {
        toggles.push((value.row, value.on));
    }
}
```

**Command dispatch wrapper** (lines 1004-1009): keep all panel actions going through it.

```rust
pub fn run_command(&mut self, command: Command) {
    match self.engine.run(command) {
        Ok(_effect) => self.status = "done".to_string(),
        Err(e) => self.status = e.to_string(),
    }
}
```

**Panel / inspector analog** (lines 2700-2737): `Inspector::new(...).target(...).header(header).show(ui, |ui| ...)`; add a UI-07 section inside `inspector()` using the existing `SectionHeader` + `PropertyGroup`/`PropertyRow` (lines 2739-2765).

**New state to add** beside `selected_layer` (`StencilApp`, lines 587-654): e.g. `open_layer_groups: BTreeSet<usize>`, `pending_layer_popup: Option<usize>`, `op_drag: Option<OperationKind>`. Follow the existing plain-field style.

**Menu action dispatch** (lines 281-289, 2059-2062): add variants to `TopMenuAction` (e.g. `AssignToLayer { layer: usize }`, `RemoveFromLayer { layer: usize }`, `SetLayerOperation { index: usize, operation: Operation }`) and match them in `handle_top_menu_action`, each calling `self.run_command(...)`.

**Catalogue analog** (lines 108-167 + 367-375): `ActiveTool::{icon,label}` and `MAT_CHOICES` are the project's established value→label/icon catalogue shape; use it for `OperationKind` (see §7).

**Menu bar integration** (lines 2231-2266): `MenuBar::new(&menus).show(ui)` reports `mb.chosen`; new actions flow through the same `chosen_action` path.

---

### 7. `apps/stencilcraft/src/operations.rs` *(new: preset catalogue + palette model)*

**Analog:** `ActiveTool` label/icon mapping (`apps/stencilcraft/src/main.rs:108-167`) and `MAT_CHOICES` (`main.rs:367-375`).

```rust
impl ActiveTool {
    pub fn icon(self) -> Icon { match self { Self::Select => Icon::Pointer, /* ... */ } }
    pub fn label(self) -> &'static str { match self { Self::Select => "Select", /* ... */ } }
}
pub const MAT_CHOICES: &[(stencilcraft_machine::Mat, &str)] = &[
    (stencilcraft_machine::Mat::None, "no mat"),
    (stencilcraft_machine::Mat::Cameo12x12, "12×12"),
];
```

Build D-17's minimal built-in catalogue in this shape (Cut / Engrave / Score / Guide; `OperationSettings::default()`; `preset_id` unset):

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum OperationPreset { Cut, Engrave, Score, Guide }
impl OperationPreset {
    pub const ALL: [Self; 4] = [Self::Cut, Self::Engrave, Self::Score, Self::Guide];
    pub fn label(self) -> &'static str { /* ... */ }
    pub fn icon(self) -> Icon { /* ... */ }
    pub fn operation(self) -> Operation {
        match self {
            Self::Cut => Operation::Cut(OperationSettings::default()),
            Self::Engrave => Operation::Engrave(OperationSettings::default()),
            Self::Score => Operation::Score(OperationSettings::default()),
            Self::Guide => Operation::Guide,
        }
    }
}
```

Also define the drag payload here (distinct type, D-13 / RESEARCH Pattern 3):

```rust
#[derive(Clone, Copy, Debug)]
pub struct OpDrag(pub OperationPreset);
```

`Operation` variants/settings are at `refs/craft-libs/crates/stencilcraft-document/src/production.rs:89-100` and `:53-69`; guide test at `:156-158`; validation at `:177+`.

---

### 8. `apps/stencilcraft/src/menus.rs` (menu definition, request-response)

**Analog:** same file's Object menu and helpers.

**Replace the TODO** (line 236) — current:

```rust
sub("Arrange", vec![
    // ...
    SEP,
    todo("Send to Layer…"),
]),
```

**Submenu helper pattern** (lines 34-78):

```rust
fn item(label: &str, action: A) -> MenuItem<A> { MenuItem::new(label, action) }
fn when(label: &str, action: A, enabled: bool) -> MenuEntry<A> {
    item(label, action).enabled(enabled).into()
}
fn sub(label: &str, children: Vec<MenuEntry<A>>) -> MenuEntry<A> {
    MenuItem::submenu(label, children).into()
}
const SEP: MenuEntry<A> = MenuEntry::Separator;
```

Build the `Send to Layer…` children dynamically from `app.engine.doc.layers` (one assign entry per layer, plus remove entries per D-06). `object(app)` already receives `&StencilApp` (line 210), so layer data is available:

```rust
let send_to_layer = sub("Send to Layer…", app.engine.doc.layers.iter().enumerate().map(|(i, layer)| {
    when(&layer.name, A::AssignToLayer { layer: i }, any)
    // plus a remove/unassign entry per layer (D-06)
}).collect());
```

**Legacy `Line Style` block** (lines 353-361) uses `A::SetStyle(CutStyle::…)`; it must be removed or repointed at layer-operation assignment (v2 has no `shape.style`). `SetStyle` handler is at `main.rs:2059-2062`.

**Panels menu** (lines 631-643) already has `item("Layers", A::ShowPanel(WorkspacePanel::Layers))`; no change needed beyond keeping the panel reachable.

**Menu popup API** if the operation popup is rendered as a `craft_ui::menu::Menu` rather than the menu bar (`~/.cargo/git/checkouts/.../8504e84/crates/ui/src/menu/popup.rs:21-46`):

```rust
let entries: Vec<MenuEntry<&str>> = vec![
    MenuItem::new("Copy", "copy").into(),
    MenuItem::new("Paste", "paste").into(),
];
let button = ui.button("Edit");
let toggle = button.clicked_by(PointerButton::Primary).then_some(SetOpenCommand::Toggle);
let shown = Menu::new(&entries).show(ui, Popup::menu(&button).open_memory(toggle));
if let Some(command) = shown.chosen { /* run it */ }
```

`MenuResponse<T>` fields at `menu/popup.rs:102-120` (`chosen`, `open`, `dismissed`).

---

### 9. `apps/stencilcraft/src/panels.rs` (controller, request-response)

**Analog:** same file's `EditAction` + `handle_edit_action`.

`EditAction` is the shared enum for menu/shortcut/panel actions (lines 69+), dispatched in one match (lines 334-533), e.g.:

```rust
pub enum EditAction { Duplicate(ScreenSide), Lock, UnlockAll, Hide, ShowAll, /* ... */ }

pub fn handle_edit_action(&mut self, action: EditAction) {
    match action {
        EditAction::Lock => { /* run_command(...) */ }
        EditAction::UnlockAll | EditAction::ShowAll => { /* ... */ }
        // ...
    }
}
```

For Send-to-Layer, either extend `EditAction` with `SendToLayer(usize)` / `RemoveFromLayer(usize)` and dispatch here, or use `TopMenuAction` directly (§6). Prefer one shared enum so keyboard/menu/panel paths stay identical (AGENTS "the automation interface can do everything a panel does").

---

### 10. `Cargo.toml` / `Cargo.lock` (config)

**Analog:** current pin block (`worktrees/stencilcraft-planning/Cargo.toml:26-45`).

```toml
craft-ui = { git = "https://github.com/aiongg/craft-libs.git", rev = "8504e84805049d3af59baacf5239315eaaa74e50" }
# ...
craft-stencilcraft-format = { git = "https://github.com/storytold/craft-libs.git", rev = "75c70fa6a786240eb853e6267e25fc445f510cd0" }
craft-stencilcraft-document = { git = "https://github.com/storytold/craft-libs.git", rev = "75c70fa6a786240eb853e6267e25fc445f510cd0" }
```

Wave 0 changes the **stencilcraft-document/format (+ raster-codecs/dxf if the Phase 1 branch moved them) rev** to the committed v2 rev and regenerates `Cargo.lock`. Keep `craft-ui` at `8504e84…` — the list/tree/Inspector APIs this phase relies on are at that rev. `[patch.crates-io]` (lines 107+) is the fallback only (D-14).

---

## Shared Patterns

### Everything is a command; dispatch after the UI pass
**Source:** `crates/engine/src/document.rs:474-509`, `apps/stencilcraft/src/main.rs:1004-1009`
**Apply to:** every new layer checkbox, colour edit, popup choice, palette drop, reorder, assign/remove.

```rust
self.run_command(Command::Document(DocumentCommand::LayerMove { from, to }));
```
Never mutate `self.engine.doc` from `layers_panel`/`inspector_panel`.

### Geometry invariance for membership edits
**Source:** anti-pattern in `crates/engine/src/document.rs:693-718`; validation in `crates/engine/tests/production.rs:1-45`.
**Apply to:** `layer.add-members`, `layer.remove-members`, `layer.set-operation`.
Only `Layer.members`/`Layer.operation` change. Test with `doc.get_shape(id).path`/`bounds()` before vs after; a second layer's membership must survive (D-07). Never drain other layers.

### Derived, never persisted: Unassigned + Pass N
**Source:** `refs/craft-libs/crates/stencilcraft-document/src/lib.rs:489-504`
**Apply to:** Layers tree bottom group, object badges, "Pass N".
Compute per frame from `unassigned_shapes()` and `production_sequence()`. Do not create a layer or enum variant to store them.

### Layer colour / visibility flows through the painter
**Source:** `crates/ui-egui/src/lib.rs:87-101, 532-550, 621-650`
**Apply to:** UI-05, canvas reflection.
`layer.visible` gates painting; `layer.color` (parsed `#rrggbb`) overrides the operation colour token; hidden/locked shape flags stay on `Shape`.

### Tree rows, toggles, reorder, drag payloads
**Source:** `~/.cargo/git/checkouts/craft-libs-1db3ee69db7fcb08/8504e84/crates/ui/src/list/`
- `ListEvent` variants: `view.rs:42-75`
- `ListOutput { events, drop_target, rows: Vec<(K, Rect)> }`: `view.rs:102-116`
- `List::tree/selection/reorder/show`: `view.rs:248-270, 367-378`
- `RowInfo::leaf/group/at_depth/section`: `behavior/list_selection.rs:8-46`
- `ListRow::{new,detail,toggle,icon,trailing_item,selected,dimmed,show}`: `row.rs:463, 666, 673, 679, 764, 777, 789, 934`
- `ToggleColumn::eye/lock` + `ToggleEdit`: `toggle.rs:169, 174, 195-202`
- `Reorder::new/accepts`: `behavior/reorder.rs:340-358`
- Drag payloads: `response.dnd_set_drag_payload(...)` (`docking/native_tests.rs:129`) or `ui.dnd_drag_source(id, payload, |ui| ...)` (`tab_strip/stories.rs:447`).

**Apply to:** all of D-09/D-10/D-11/D-13. Use a distinct `OpDrag` payload; hit-test `ListOutput.rows` on release and dispatch `LayerSetOperation` for the row under the pointer.

### Inspector composition
**Source:** `~/.cargo/git/checkouts/.../crates/ui/src/patterns/inspector.rs:71-95, 174-249`
**Apply to:** UI-07 effective-operation display. Copy the existing usage at `apps/stencilcraft/src/main.rs:2700-2737`; add a `SectionHeader` + read-only `PropertyRow`s inside `inspector()`.

### Input validation / no panic
**Source:** `layer_index`/`editable_ids` (`crates/engine/src/document.rs:624-648`), `parse_hex_color`/`format_hex_color` (`document.rs:750-762`), `Operation::validate` (`refs/.../production.rs:177+`).
**Apply to:** all commands and all drag/drop hit tests. Treat drops, command JSON and colour strings as untrusted; reject locked-layer edits; never `unwrap` outside tests (worktree clippy bans it).

### Cross-repo paths in plans
**Source:** RESEARCH Pitfall 7.
**Apply to:** Wave 0 and any worktree command. Use absolute paths or `--manifest-path /home/r2d2/Sandbox/lasercraft/worktrees/stencilcraft-planning/Cargo.toml`; the `rustfactor` git repo cannot validate worktree files.

---

## Wave 0 Migration Map (v1 → v2)

Mechanical mirror: the analog is the v2 model itself plus the current v1 call site. Field mapping:

| v1 (worktree pin `75c70fa`) | v2 (Phase 1) |
|---|---|
| `Layer.kind: CutStyle` | `Layer.operation: Option<Operation>` (`CutStyle::Cut` → `Operation::Cut(default)`, `Engrave` → `Operation::Engrave(default)`, `None` for container) |
| `Layer.template: bool` | `Layer.operation = Some(Operation::Guide)` |
| `Layer.shapes: Vec<Shape>` | `Layer.members: Vec<u64>` + `Document.shapes` |
| `Shape.style: CutStyle` | removed — resolve the shape's layer/operation via membership |
| `ShapeProduction.operation: Option<Operation>` | removed — layer operation (per-shape tool op → membership) |
| `Document::shapes()` yields `(usize, &Shape)` | yields `&Shape` |
| `doc.layers[i].shapes.iter()` | `doc.layers[i].members.iter().filter_map(|id| doc.get_shape(*id))` |
| `Layer::new(name, CutStyle)` | `Layer::new(name, Option<Operation>)` |

Call-site inventory (counts above are raw occurrences incl. tests; recount in Wave 0):

| File | ~occurrences | Notes |
|---|---:|---|
| `crates/engine/src/lib.rs` | 117 | largest; command handlers, session sync, replicated shapes |
| `crates/svg/src/lib.rs` | 59 | export/import walks layers |
| `crates/doc/src/templates.rs` | 50 | built-in templates construct `Layer`/`Shape` |
| `crates/engine/src/document.rs` | 25 | layer commands |
| `crates/kerf/src/lib.rs` | 23 | layer/shape iteration |
| `crates/import/src/handlers.rs` | 23 | imported paths → layers |
| `apps/stencilcraft/src/main.rs` | 23 | panels, selection, layer data |
| `crates/engine/src/generate.rs` | 21 | generators |
| `crates/testkit/src/lib.rs` | 20 | sample docs |
| `crates/tiling/src/lib.rs` | 18 | tiling walks shapes |
| `crates/dxf/src/lib.rs` | 18 | export layer grouping |
| `crates/gcode/src/lib.rs` | 15 | export layer grouping |
| `crates/ui-egui/src/lib.rs` | 12 | paint/hit-test |
| `crates/engine/src/text.rs` | 12 | `Layer::new(name, style)` at :192 |
| `crates/import/src/lib.rs` | 11 | |
| `crates/layout/src/mats.rs`, `crates/layout/src/job.rs`, `crates/layout/src/print.rs` | 10/7/4 | job/mat layering |
| `crates/engine/src/production.rs` | 9 | per-shape operation writers |
| `crates/pdf/src/lib.rs`, `crates/import/src/render.rs` | 7/6 | |
| `crates/studio/src/lib.rs`, `crates/studio/examples/studio_dump.rs` | 4/2 | |
| `crates/mcp/src/lib.rs` | 1 | command surface |
| `crates/engine/src/hardware.rs` | 1 | |
| Tests | — | `crates/engine/tests/*`, `crates/svg/tests/*`, `crates/import/tests/*`, `crates/studio/tests/*`, `apps/stencilcraft-cli/tests/e2e.rs` |

Compile gate: `cargo check --workspace --all-targets` from the worktree, then `cargo xtask ci`.

---

## Test Pattern Assignments

### Engine unit tests — `crates/engine/src/document.rs` `mod tests`
**Analog:** lines 1930-1960 (`three()`, `run()`, `bounds()`, `order()`).

```rust
fn three() -> Engine {
    let mut doc = Document::default();
    for x in [0.0, 20.0, 40.0] {
        doc.push_shape(0, rect(x, 0.0, 10.0, 10.0), CutStyle::Cut).unwrap();
    }
    Engine::new(doc)
}
fn run(engine: &mut Engine, command: DocumentCommand) -> Effect {
    engine.run(Command::Document(command)).unwrap()
}
```

Add tests required by RESEARCH Validation Architecture:
- `layer_add_is_operationless` — `LayerAdd` yields `operation: None`.
- `document_new_has_no_cut` — `DocumentNew` default layer is `("base", None)`.
- `assign_preserves_geometry` — add/remove members leaves every `doc.shapes` entry's `path`/`bounds` equal and keeps other memberships.
- `reorder_changes_pass_order` — `LayerMove` changes `production_sequence()` / Pass N.
- `layer_properties_roundtrip` — colour/visible/lock via commands; invalid `#hex` rejected.
- `commands_round_trip_through_the_command_json` — extend the sample array at lines 1962-2004 with the three new variants.

### Engine integration tests — `crates/engine/tests/production.rs`
**Analog:** lines 1-45 (JSON envelope + geometry assertions).

```rust
fn run(engine: &mut Engine, json: serde_json::Value) -> Result<Effect, String> {
    let command: Command = serde_json::from_value(json).map_err(|e| e.to_string())?;
    engine.run(command).map_err(|e| e.to_string())
}
```
Use this for the serde-tagged dot names (`"layer.add-members"` etc.) and undo round-trips.

### UI pure-helper tests — `crates/ui-egui/src/lib.rs` `mod tests`
**Analog:** lines 1296+; `kerf_preview_changes_visuals` at 1539-1547.
Add `layer_colour_affects_visual`, `unassigned_rows`, `effective_operation`, `pass_number_skips_container_layers`.

### Model tests — `refs/craft-libs/.../tests/it/main.rs`
Only if membership helpers are promoted into `craft-stencilcraft-document`. The Phase 1 model tests already cover `unassigned_shapes`/`production_sequence` (RESEARCH: model test exists); do **not** duplicate library behaviour (craft-libs AGENTS rule 27).

---

## No Analog Found

| File / concern | Role | Data Flow | Reason / fallback |
|---|---|---|---|
| Colour picker swatch | component | request-response | `craft-ui` has no colour-picker widget at rev `8504e84…`; app has no analog. Use egui `color_edit_button_srgba` (RESEARCH A5) + `parse_hex_color`/`format_hex_color` from `stencilcraft_doc::canvas`. |
| Operation popup anchored to the Add-Layer shift+click | component | request-response | No app usage of `menu::Menu` yet. Use the `Menu` doc example (§8) with a configurable modifier check; source of truth is `menu/popup.rs:21-46`. |
| v1→v2 migration itself | mixed | mixed | No previous cross-repo migration exists. Use the field map above + v2 model source; this is Wave 0, not a pattern to copy. |
| `[patch]` fallback wiring | config | n/a | Only needed if the Phase 1 branch cannot be pushed (D-14); no local precedent. |

---

## Metadata

**Analog search scope:** `worktrees/stencilcraft-planning/{crates/engine,crates/ui-egui,crates/doc,crates/layout,crates/gcode,crates/mcp,crates/studio,crates/dxf,apps/stencilcraft,apps/stencilcraft-cli}`, `refs/craft-libs/crates/stencilcraft-document`, and the resolved Cargo checkout for `craft-ui` rev `8504e84…`.
**Files scanned:** ~40 source files + 5 model/UI-library files.
**Pattern extraction date:** 2026-11-24.

**Notes for the planner**
- Prefer inline `mod tests` in `crates/engine/src/document.rs` (already the project's layer-test home) over a new test binary; the crate already has `tests/production.rs` for envelope tests.
- `craft_ui::list::List` is available at the pinned rev but **not yet used anywhere in the app**; the first tree integration is this phase. Copy the API shape from `view.rs`/`row.rs`, not from a previous app panel.
- `ShapesMoveToLayer` and `ShapesSetStyle` are v1 leftovers: never reuse them for assignment; remove/repoint them during Wave 0/UI work.
- Keep the `craft-ui` rev unchanged; bump only the `craft-stencilcraft-*` rev (and its dependent `craft-*` crates if the Phase 1 branch moved them), then regenerate `Cargo.lock`.
