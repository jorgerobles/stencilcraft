# Phase 1: Operation-First Document Model - Pattern Map

**Mapped:** 2026-11-24
**Files analyzed:** 8 (Phase 1 lands in `refs/craft-libs` only; consumer adaptation is Phase 2/3 — see bottom)
**Analogs found:** 8 / 8

> **Where the code lives.** All Phase-1 source changes land in the pristine `refs/craft-libs` clone (`main @ 249a41b`, workspace `0.1.0`). Every analog below is git-tracked source in that repo (verified with `git ls-files`); none is a build/install mirror. The StencilCraft worktree `worktrees/stencilcraft-planning` is a **reference only** for Phase 1 — its crates are adapted in Phase 2/3 (RESEARCH Assumption A1 / Open Q1–Q2).

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `crates/stencilcraft-format/src/lib.rs` (MODIFY) | codec / wire schema | transform (parse→typed, typed→JSON) | itself (current v1 envelope) | exact (self-evolution) |
| `crates/stencilcraft-document/src/lib.rs` (MODIFY) | model + validation | CRUD (document state) | itself (`Document`/`Layer`/`Shape`/`save_json`/`load_json`) | exact (self-evolution) |
| `crates/stencilcraft-document/src/production.rs` (MODIFY) | model (enums) | transform (v1 op → v2 cooked op) | itself (`Operation`, `ToolOp`, `Shape::operation`) | exact (self-evolution) |
| `crates/stencilcraft-document/src/migrate.rs` (CREATE) | service (pure migration fn) | batch / transform | `psd::abr::parse` version dispatch + `Document::pruned_groups` prune loop | role-match |
| `crates/stencilcraft-document/src/wire_v1.rs` (CREATE) | model (read-only wire types) | transform (deserialize only) | `craft-stencilcraft-format::DocumentFields` + `ShapeProduction` serde-defaults | role-match |
| `crates/stencilcraft-document/src/canvas.rs` (MODIFY) | model helpers | CRUD (queries / projections) | itself (`pruned_groups`, `page_document`, `render_thumbnail`) | exact (self-evolution) |
| `crates/stencilcraft-document/tests/it/main.rs` (CREATE) | test | — | `crates/material-graph/tests/it/main.rs` (mod-per-feature binary) | exact (rule 30) |
| `Cargo.toml` + `CHANGELOG.md` (MODIFY) | config / docs | — | current `[workspace.package]` + `## v0.1.0` entry | exact |

**Not modified (reused as-is):** `crates/stencilcraft-document/src/geom.rs` — `check_finite_mm` (`:44-47`) and `bounds_of_shape` (`:171`) are called, not changed.

## Pattern Assignments

### `crates/stencilcraft-format/src/lib.rs` (codec, transform)

**Analog:** itself — current v1 envelope. This is the file that blocks v2 (strict version check + field whitelist). Copy its existing structure and extend it; do not invent a new codec.

**Constants pattern** (lines 29-36):
```rust
pub const EXTENSION: &str = "stencilcraft";
pub const FORMAT_NAME: &str = "stencilcraft";
pub const FORMAT_VERSION: u64 = 1;   // → 2
pub const HEADER_KEY: &str = "stencilcraft";
```
Add `pub const SUPPORTED_VERSIONS: &[u64] = &[1, 2];` (RESEARCH Pattern 1) and keep `FORMAT_VERSION = 2` as the write target.

**Version-dispatch pattern to change** (lines 175-189, `from_typed_wire`) — the hard blocker (Pitfall 1):
```rust
let Some(version) = header.get("version").and_then(Value::as_u64) else {
    return Err(Error::NotStencilCraft);
};
if version != FORMAT_VERSION {                       // ← replace with range check
    return Err(Error::Version { found: version, supported: FORMAT_VERSION });
}
let raw: RawFile<T> = serde_json::from_value(Value::Object(wire.fields))
    .map_err(|source| Error::Document { source })?;
Ok(Self { document: raw.document })
```
Target: accept `1..=2`, return the detected `version` to the caller, and let the document crate dispatch `1 => migrate_v1`, `2 => parse_v2`. Keep the old-build rejection path intact for a future `3` (existing `Error::Version` semantics).

**Field-whitelist pattern to extend** (lines 312-329) — Pitfall 2, silently drops unlisted top-level fields:
```rust
fn is_wire_field(name: &str) -> bool {
    matches!(name,
        HEADER_KEY | "board" | "kerf" | "layers" | "next-shape-id" | "images"
        | "board-setup" | "pages" | "guides" | "grid" | "snap" | "groups" | "meta")
}
```
Add `"shapes"` (and any other v2-only top-level name). Add a round-trip test asserting `shapes` survives save→load.

**Error taxonomy to reuse** (lines 345-361): `Error::{Json, NotStencilCraft, Version{found,supported}, Document{source}}` — extend, never replace.

**Test pattern to mirror** (lines 390-402, 432-435, 491-511): round-trip through `File::read`/`read_typed` and assert `Version { found, supported }` on a future version. Adapt `refuses_versions_other_than_the_current_version` to the new `1..=2` contract (v1 accepted, v3 rejected).

---

### `crates/stencilcraft-document/src/lib.rs` (model, CRUD)

**Analog:** itself — `Document`/`Layer`/`Shape`/`save_json`/`load_json` are all in this one file. Extend in place.

**Error type to extend** (lines 23-45) — extend `DocError`, do not add a new error enum:
```rust
#[derive(Debug, thiserror::Error)]
pub enum DocError {
    #[error("{path}: {source}")] Io { path: String, source: std::io::Error },
    #[error("{path}: not a stencilcraft file")] NotAStencilcraftFile { path: String },
    #[error("{path}: format version {found}, this build reads {expected}")]
    FormatVersion { path: String, found: u64, expected: u64 },
    #[error("{path}: {source}")] Parse { path: String, source: serde_json::Error },
    #[error("{path}: {message}")] Structure { path: String, message: String },
    #[error("document: {0}")] Model(String),
    #[error(transparent)] Geom(#[from] GeomError),
}
```

**v1 `Layer` to replace** (lines 87-135) — v2 removes `kind` and `shapes`, adds `operation: Option<Operation>` + `members: Vec<u64>`. Keep `color: Option<String>` (`#rrggbb`), `locked`, `visible`, `name`, and the `skip_serializing_if` discipline:
```rust
pub struct Layer {
    pub name: String,
    pub kind: CutStyle,                 // legacy v1 only — removed from v2 type
    pub visible: bool,
    pub shapes: Vec<Shape>,             // removed from v2 — geometry hoisted to Document
    #[serde(default, skip_serializing_if = "is_false")] pub locked: bool,
    #[serde(default, skip_serializing_if = "is_false")] pub template: bool,
    #[serde(default, skip_serializing_if = "Option::is_none")] pub color: Option<String>,
}
```

**v1 `Shape` to keep (geometry unchanged)** (lines 137-161): keep `id: u64`, `path: BezPath`, `hidden`, `locked`, `text`; `style`/`tool`/`production` become legacy read-only (D-16). `Shape` moves out of `Layer` into `Document.shapes`.

**`Document` to extend** (lines 381-410) — add `pub shapes: Vec<Shape>` beside `pub layers: Vec<Layer>`:
```rust
pub struct Document {
    pub board: Board,
    pub kerf: KerfSettings,
    pub layers: Vec<Layer>,
    #[serde(default)] pub next_shape_id: u64,
    #[serde(default)] pub images: Vec<RasterImage>,
    #[serde(default)] pub board_setup: PageSetup,
    // … pages, guides, grid, snap, groups, meta unchanged …
}
```

**`Default` to change** (lines 428-445) — Pitfall 6. Replace `layers: vec![Layer::new("cut", CutStyle::Cut)]` with one base layer `operation: None` (D-18). Add a `Layer::new(name, Option<Operation>)`-style constructor.

**Shape accessors to rework** (lines 447-492) — `shapes()` / `shape_count()` / `get_shape` / `get_shape_mut` currently walk `layer.shapes`; re-point them at `self.shapes` (MODEL-01). `push_shape` (467-479) currently `layer.shapes.push(...)`; change it to push into `self.shapes` and record membership in the target layer:
```rust
pub fn push_shape(&mut self, layer_index: usize, path: BezPath, style: CutStyle) -> Result<Shape, DocError> {
    if layer_index >= self.layers.len() { /* DocError::Model … */ }
    let id = self.next_shape_id;
    self.next_shape_id = self.next_shape_id.checked_add(1)
        .ok_or_else(|| DocError::Model("shape ids exhausted".to_string()))?;
    // push Shape into self.shapes; self.layers[layer_index].members.push(id)
}
```
Keep the id allocator and its overflow error verbatim.

**`validate` to extend** (lines 498-514) — add: unique shape ids, every `members` id exists (or prune like `pruned_groups`), and finite checks on every new numeric operation setting (Pitfall 8):
```rust
pub fn validate(&self) -> Result<(), DocError> {
    check_finite_mm(self.board.width_mm, "board width")?;
    check_finite_mm(self.board.height_mm, "board height")?;
    check_finite_mm(self.kerf.width_mm, "kerf width")?;
    if self.board.width_mm <= 0.0 || self.board.height_mm <= 0.0 { /* … */ }
    self.images.iter().try_for_each(RasterImage::validate)?;
    self.validate_canvas()
    // + v2: unique ids, membership referential integrity, finite operation params
}
```

**save/load dispatch pattern** (lines 519-574) — this is the core change:
```rust
pub fn save_json(&self) -> Result<String, DocError> {
    self.validate()?;
    let fields = /* v2 DocumentFields incl. shapes, members */;
    craft_stencilcraft_format::File::<NativeFileFields>::new(fields).to_json_pretty()
        .map_err(|e| DocError::Structure { path: "output".to_string(), message: e.to_string() })
}

fn load_json_named(json: &str, path: &str) -> Result<Document, DocError> {
    let (version, fields) = /* format crate read_versioned */;
    match version {
        1 => migrate_v1(parse_v1(fields)?, &MigrationSeed::default()),
        2 => parse_v2(fields)?,
        _ => /* already rejected by format crate */,
    }
    // then doc.validate()?; doc.groups = doc.pruned_groups();
}
```
Keep the `map_err` chain that turns `craft_stencilcraft_format::Error` into calm `DocError` variants (546-555). Keep `save_to` on `craft_atomic::atomic_write` (581-584).

**New methods to add** (RESEARCH Pattern 3 / MODEL-05, MODEL-07):
```rust
pub fn unassigned_shapes(&self) -> impl Iterator<Item = &Shape> {
    let assigned: BTreeSet<u64> = self.layers.iter()
        .filter(|l| l.operation.is_some())
        .flat_map(|l| l.members.iter().copied()).collect();
    self.shapes.iter().filter(move |s| !assigned.contains(&s.id))
}
pub fn production_sequence(&self) -> impl Iterator<Item = &Layer> {
    self.layers.iter().filter(|l| l.operation.is_some())
}
pub fn remove_layer(&mut self, index: usize) -> Result<(), DocError> {
    // drop the Layer (with its members) only — never touch self.shapes (MODEL-07)
}
```

**Test pattern to mirror** (lines 613-619, 682-695, 776-787): `round_trip` (save→load→`assert_eq!`), `future_version_is_rejected_calmly`, and the fixture loop `version_1_files_without_canvas_fields_still_load` — this loop is the direct template for the migration tests (reuse the same `include_str!("../tests/fixtures/…")` files).

---

### `crates/stencilcraft-document/src/production.rs` (model, transform)

**Analog:** itself — `Operation` (9 variants), `ToolOp`, `Shape::operation()` already encode the v1→effective-pass precedence the migration needs.

**v1 `Operation` (wire identity to preserve)** (lines 12-34): keep all nine kebab-case tags so v2 cooked variants round-trip:
```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Serialize, Deserialize)]
#[serde(rename_all = "kebab-case")]
pub enum Operation {
    Cut, KissCut, DieCut, Draw, Score, Engrave, Perforate, PrintThenCut, Guide,
}
```

**`Shape::operation()` — the exact fallback precedence for D-14** (lines 199-205):
```rust
pub fn operation(&self) -> Operation {
    self.production.operation.unwrap_or(match self.style {
        crate::CutStyle::Cut => Operation::Cut,
        crate::CutStyle::Engrave => Operation::Engrave,
    })
}
```
Migration precedence is `Shape.tool` (ToolOp) → `ShapeProduction.operation` → this style fallback.

**`ToolOp` variants to map into cooked operations** (lines 179-205): `Score{tip,passes}`→Score; `Emboss{tip,passes,deboss}`→Engrave/emboss; `PowerEngrave`→Engrave; `HeatPen`→Print/foil; `Punch{diameter_mm}`→Punch. Carry `tip`/`passes`/`deboss`/`diameter_mm` into the layer settings snapshot.

**v2 `Operation` representation (recommended, RESEARCH Standard Stack → Operation)** — add a data-carrying enum + settings struct in this module, keep process/strategy as derived accessors (D-06):
```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "kebab-case")]
pub enum Operation {
    Cut(LaserParams), Engrave(LaserParams), Score(ScoreParams), KissCut(LaserParams),
    DieCut(LaserParams), Draw(PenParams), Perforate(LaserParams),
    PrintThenCut(PrintParams), Guide,
}
impl Operation {
    pub fn process(&self) -> Process { /* laser | mill | knife | pen | … */ }
    pub fn strategy(&self) -> Strategy { /* line | fill | pocket | score | … */ }
}
```
Settings struct carries the D-09 snapshot: `power`, `speed_mm_per_min`, `passes`, `focus_mm`, `kerf_mm`, `tabs`, strategy params — **all finite-checked**. Preserve `ShapeProduction` (140-174) and `Shape::cut_path` (176-198) unchanged; they are legacy-read + geometry concerns.

---

### `crates/stencilcraft-document/src/migrate.rs` (CREATE — service, batch/transform)

**Analog:** `crates/psd/src/abr.rs::parse` (version dispatch) + `crates/stencilcraft-document/src/canvas.rs::pruned_groups` (deterministic set prune).

**Version-dispatch skeleton to copy** — `crates/psd/src/abr.rs:494-503`:
```rust
pub fn parse(data: &[u8]) -> Result<AbrFile> {
    let version = r.u16()?;
    let mut out = AbrFile { version, ..Default::default() };
    match version {
        1 | 2 => parse_v12(&mut r, version, &mut out)?,
        6..=10 => parse_v6(&mut r, &mut out)?,
        v => return Err(PsdError::Unsupported(format!("brush file version {v}"))),
    }
    Ok(out)
}
```
Same shape for `migrate_v1(v1, seed)`: a pure function, no I/O, `Result`-returning, one arm per version handled.

**Deterministic-order pattern to copy** — `canvas.rs::pruned_groups` (lines 424-443) uses `BTreeSet`/`BTreeMap` for stable, ordered id maths:
```rust
pub fn pruned_groups(&self) -> Vec<Group> {
    let shapes: BTreeSet<u64> = self.shapes().map(|(_, s)| s.id).collect();
    // …retain/retain_until-fixpoint…
}
```
Use `BTreeSet<u64>` for membership set maths (unassigned/dangling) but **`Vec<u64>` for `members`** — order is semantic (Pitfall 4).

**Migration body (RESEARCH Pattern 2)** — grouping key = effective pass, preserve layer order and intra-layer shape order:
```rust
// Precedence (D-14): Shape.tool > ShapeProduction.operation > Shape.style fallback.
for v1_layer in v1.layers {                       // preserve original layer order
    let mut groups: Vec<(PassKey, Vec<u64>)> = Vec::new(); // IndexMap not available — small layers
    for shape in v1_layer.shapes {                // preserve intra-layer order (D-20)
        hoist_shape_once(&mut doc.shapes, &shape);        // geometry stored once (D-02)
        let key = effective_pass(&shape);
        match groups.iter_mut().find(|(k, _)| *k == key) {
            Some((_, members)) => members.push(shape.id),
            None => groups.push((key, vec![shape.id])),
        }
    }
    for (pass, members) in groups {
        doc.layers.push(Layer { operation: Some(pass.into_operation(seed)), members, .. });
    }
}
```
Define `pub struct MigrationSeed { … }` (neutral `OperationDefaults`) in this crate so `load_json` uses `Default` and a future `load_json_with_seed` takes the app's mapped `GcodeSettings` — never `use stencilcraft_gcode::…` here (Pitfall 5, layer violation).

---

### `crates/stencilcraft-document/src/wire_v1.rs` (CREATE — model, deserialize-only)

**Analog:** `craft-stencilcraft-format::DocumentFields` (serde defaults) + `ShapeProduction`/`TextSpec` default discipline in `lib.rs`.

**Serde-default discipline to copy** — `lib.rs:96-103, 145-160` and `format/lib.rs:66-92`: `#[serde(default)]`, `#[serde(rename = "next-shape-id")]`, `#[serde(skip_serializing_if = "is_default")]`, `#[serde(default = "one_pass")]`. The v1 types must **deserialize only** (no `Serialize`) so legacy fields can never be written into v2:
```rust
#[derive(Debug, Deserialize)]              // Deserialize only — deliberately not Serialize
pub struct V1Layer { pub name: String, pub kind: CutStyle, pub visible: bool,
                     pub shapes: Vec<V1Shape>, /* locked/template/color */ }
```
Mirror the v1 wire shape exactly as seen in `tests/fixtures/starter.stencilcraft` (`"kind": "cut"`, inline `"shapes"`, `"style": "cut"`, header `"version": 1`).

---

### `crates/stencilcraft-document/src/canvas.rs` (MODIFY — model helpers, CRUD queries)

**Analog:** itself. Every helper that walks `layer.shapes` must move to `self.shapes` + `layer.members`.

**`pruned_groups`** (lines 424-443): change `self.shapes()` source to the document-level store; the fixpoint/retain loop is unchanged.

**`shape_locked` / `layer_of_shape`** (lines 375-382): re-implement via membership — `layer_of_shape` returns the first layer whose `members` contains `id`; `shape_locked` checks `self.get_shape(id).locked || any holding layer.locked`.

**`page_document`** (lines 468-484): currently `layer.shapes.retain(...)` and `shape.path.apply_affine(shift)` per layer. Rework to filter `out.shapes` by `page_of_shape`, translate geometry once, then prune `members` to surviving ids:
```rust
out.shapes.retain(|s| self.page_of_shape(s) == index);
for shape in &mut out.shapes { shape.path.apply_affine(shift); }
for layer in &mut out.layers { layer.members.retain(|id| /* id still in out.shapes */); }
out.groups = out.pruned_groups();
```

**`render_thumbnail`** (lines 488-520): iterate `self.production_sequence()` (or `is_output()` layers) → resolve `members` → `get_shape(id)` → write path. Keep the `craft_svg::write::SvgWriter` budget loop verbatim.

**`group_shapes`** (lines 407-421): keep the `BTreeSet` traversal; it already works off `self.shapes()` ids once that accessor is re-pointed.

---

### `crates/stencilcraft-document/tests/it/main.rs` (CREATE — test)

**Analog:** `crates/material-graph/tests/it/main.rs` (16 lines) — the canonical rule-30 shape:
```rust
//! The graph runtime's integration tests in one executable, with one module per feature.
mod deep;
mod determinism;
mod format_doc;
mod func;
// …
```
Copy that: one binary, one `mod` per behaviour. Suggested modules matching the RESEARCH test map:
```rust
//! Phase-1 integration tests: one module per behaviour (craft-libs rule 30).
mod fixtures;      // shared include_str! v1 fixtures
mod migrate;       // MODEL-04: v1 fixtures → v2, geometry + production preserved
mod round_trip;    // MODEL-01/02/03/06: save→load equality, N:M membership, all 9 ops
mod unassigned;    // MODEL-05: derived unassigned set, never persisted
mod remove_layer;  // MODEL-07: drops memberships, keeps geometry
```
Reuse the existing v1 fixtures (`tests/fixtures/{starter,poster,stencil-frame}.stencilcraft`) via `include_str!` — the `lib.rs:776-787` test already proves they load. Add `tests/it/main.rs` (and submodules) as the only new test files; do not add an external test crate.

---

### `Cargo.toml` + `CHANGELOG.md` (MODIFY — config/docs)

**Analog:** current `[workspace.package]` (`Cargo.toml:6-10`: `version = "0.1.0"`, `edition = "2024"`, `rust-version = "1.95"`) and the `## v0.1.0 — 2026-10-12` entry in `CHANGELOG.md`.

Per `docs/versioning.md` a persisted-representation change is a **minor** bump: `0.1.0 → 0.2.0`. Add a `## v0.2.0` section explaining before/after and the v1→v2 migration, mirroring the existing entry's structure. Both crate `Cargo.toml`s already have `[lints] workspace = true` (verified) — no change needed there.

## Shared Patterns

### Serde defaults + omission
**Source:** `crates/stencilcraft-document/src/lib.rs:96-103,145-160`; `crates/stencilcraft-format/src/lib.rs:66-92`
**Apply to:** every new/modified struct in `lib.rs`, `production.rs`, `wire_v1.rs`, `format/lib.rs`.
```rust
#[serde(default, skip_serializing_if = "is_false")] pub locked: bool,
#[serde(default, skip_serializing_if = "Option::is_none")] pub color: Option<String>,
#[serde(rename = "next-shape-id", default = "default_next_shape_id")] pub next_shape_id: u64,
```
New optional v2 fields must be `skip_serializing_if` so a plain document saves as before; new required v1-compat fields get `default`.

### Calm `Result` errors, never panic
**Source:** `crates/stencilcraft-document/src/lib.rs:23-45` (`DocError`), `:546-578` (format→domain `map_err` chain); crate lint header `format/src/lib.rs:1-2` (`deny(unwrap_used, expect_used, panic, …)`, `forbid(unsafe_code)`).
**Apply to:** all files. Non-test code returns `Result`; extend `DocError`/`format::Error` rather than adding new enums; keep messages printable and path-prefixed. `#![deny(clippy::unwrap_used, clippy::expect_used, clippy::panic, clippy::unimplemented, clippy::todo, clippy::unreachable)]` already guards the format crate — match it in new modules.

### Finite-mm validation
**Source:** `crates/stencilcraft-document/src/geom.rs:44-47` (`check_finite_mm`), used in `lib.rs:500-514`.
**Apply to:** every new numeric operation setting (power/speed/focus/kerf/tabs) in `production.rs`, validated from `Document::validate`. Reject NaN/Inf/negative before save (Pitfall 8, ASVS V5).

### Atomic file I/O
**Source:** `crates/stencilcraft-document/src/lib.rs:581-589` (`save_to` → `craft_atomic::atomic_write`).
**Apply to:** the save path; do not add `fs::write`. Unchanged contract.

### Deterministic ordering
**Source:** `crates/stencilcraft-document/src/canvas.rs:424-443` (`BTreeSet`/`BTreeMap` prune).
**Apply to:** membership set maths (use `BTreeSet` for "assigned"/"unassigned" computation) **but** `Layer.members` stays `Vec<u64>` because intra-layer order is semantic (D-20, Pitfall 4).

### Versioned dispatch + old-build rejection
**Source:** `crates/stencilcraft-format/src/lib.rs:175-189` (strict check) and `crates/psd/src/abr.rs:494-503` (range `match`).
**Apply to:** `format::from_typed_wire`/`from_wire` (accept `1..=2`, surface detected version) and `Document::load_json_named` (`match version`).

### Integration-test layout
**Source:** `crates/material-graph/tests/it/main.rs:1-16`; `refs/craft-libs/AGENTS.md` rule 30.
**Apply to:** `crates/stencilcraft-document/tests/it/main.rs` — one binary, one module per behaviour.

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| *(none)* | — | — | Every Phase-1 file has a same-role analog in the existing crates; migration logic adapts `psd::abr` dispatch + `pruned_groups` prune. |

## Out of Scope — Phase 2/3 consumer touchpoints (reference only)

These are **not** Phase-1 edits (RESEARCH A1). Listed so the planner tracks the rev-bump dependency; they currently read v1 fields and will break when the document crates bump to `0.2.0`.

| Consumer file | Current pattern | Phase-1 dependency |
|---------------|-----------------|--------------------|
| `worktrees/stencilcraft-planning/crates/gcode/src/lib.rs` | `GcodeSettings` (`:24-30`), `export` reads `layer.kind` (`:76-130`) | Phase 3 reads layer `operation` + settings; `GcodeSettings` seeds `MigrationSeed` |
| `worktrees/stencilcraft-planning/crates/engine/src/lib.rs` | `LayerRemove` does `doc.layers.remove(index)` + "keep ≥1 layer" (`:1315-1327`) | Phase 2 wraps `Document::remove_layer` (membership-only delete) |
| `worktrees/stencilcraft-planning/crates/engine/src/document.rs` | `shape.style = l.kind` (`:714`) | Phase 2 → layer-operation assignment |
| `worktrees/stencilcraft-planning/crates/layout/src/job.rs` | `shape.operation()` naming (`:231-244`), grouping (`:428`) | Phase 3 → derived from layer operation |
| `worktrees/stencilcraft-planning/crates/layout/src/mats.rs` | `shape.operation()` grouping (`:188-206`) | Phase 3 → layer operation / derived view |
| `worktrees/stencilcraft-planning/crates/doc/src/templates.rs` | `Layer::new(name, CutStyle)`, `doc.layers[i].shapes` (`:359-374,461-475`) | Phase 2 → `Layer::new(name, Option<Operation>)` + `push_shape` |

**Wave-0 dev wiring (setup, not source):** create a local craft-libs branch/worktree off `refs/craft-libs` and add a `[patch]`/branch rev so the StencilCraft workspace can compile against v2 (RESEARCH Runtime State Inventory; Open Q1). The `refs/craft-libs` clone is currently pristine on `main @ 249a41b` (only untracked `.codegraph/`).

## Metadata

**Analog search scope:** `refs/craft-libs/crates/{stencilcraft-document,stencilcraft-format,psd,material-graph}` (tracked source); consumer reference `worktrees/stencilcraft-planning/crates/{gcode,engine,layout,doc}`.
**Files scanned:** 12 source + 3 fixtures + 2 manifests + `CHANGELOG.md` + `docs/versioning.md`.
**Pattern extraction date:** 2026-11-24
**Tracked-source check:** every analog path is git-tracked in `refs/craft-libs` / the consumer worktree; no gitignored mirror paths emitted.
