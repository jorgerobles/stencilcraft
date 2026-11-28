# Phase 1: Operation-First Document Model - Research

**Researched:** 2026-11-24
**Domain:** Rust document schema evolution (craft-libs `craft-stencilcraft-document` + `craft-stencilcraft-format`) with in-memory v1→v2 migration
**Confidence:** HIGH on the existing model and migration mechanics (read from source this session); MEDIUM on the exact v2 `Operation` settings schema (explicitly the agent's discretion); LOW/ASSUMED on cross-repo execution wiring (where Phase 1 lands).

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

#### Core Model

- **D-01:** A layer is a container. `Layer.operation` is optional, like Illustrator's appearance marker: `Some(operation)` means the layer is a production pass; `None` means it does not produce and does not enter the job sequence. Any number of layers may have no operation. — **Reversibility:** one-way — undoing this changes the v2 schema and every consumer of `Layer`.
- **D-02:** Shape geometry is stored once at document level. Layers reference shapes; a shape can belong to zero, one, or many operation layers (N:M). — **Reversibility:** one-way — undoing this needs a document migration and changes every shape/layer API.
- **D-03:** A layer with `Some(operation)` includes all of its shapes in that pass. Assigning an operation to an existing operation-less layer includes all shapes already in it. — **Reversibility:** costly — undo requires changing assignment semantics and possibly splitting layers.
- **D-04:** "Unassigned" is derived, never persisted. It is exactly the set of shapes with no membership in any layer that has `Some(operation)`. A shape in an operation-less container is unassigned unless it also belongs to an operation layer. — **Reversibility:** costly — undo touches queries, UI grouping, and output filters.
- **D-05:** Direct object operation is still a one-shape operation layer; it does not add a per-shape production authority. — **Reversibility:** costly — undo reintroduces per-shape operation ownership.

#### Operation Semantics

- **D-06:** Internally, an operation has two axes: process/tool (laser, mill, knife, pen, creaser, punch, print) and strategy (line, fill, pocket, drill, score, kiss-cut, die-cut, perforate, guide), plus parameters. The user does not edit raw axes. — **Reversibility:** costly — undo changes the operation schema and preset catalogue.
- **D-07:** The user-facing unit is a cooked, parametric "New operation" preset (Laser cut, Laser engrave fill, Score, Punch, Mill profile, Mill pocket, Kiss-cut, Die-cut, Draw, Guide, etc.). Presets live in the profiles system and can be duplicated or customised. — **Reversibility:** costly — undo changes the profile and operation model.
- **D-08:** `ToolOp` is not a second production authority. It migrates into the cooked operation model. `Shape.tool` and `ShapeProduction.operation` become legacy read-only fields. — **Reversibility:** costly — undo would restore two overlapping operation sources.
- **D-09:** Phase 1 includes the complete per-operation settings snapshot: power, speed, passes, focus, kerf, tabs, and the process/strategy-specific parameters. Emission of those settings remains Phase 3. — **Reversibility:** costly — undo requires a schema change and a second migration.
- **D-10:** Every layer embeds its operation snapshot. `preset_id` is optional provenance, not a dependency. A document must remain usable without the originating profile library. — **Reversibility:** one-way — undo changes the v2 schema and portability contract.
- **D-11:** Profiles follow LightBurn's Assign/Link model. **Assign** copies settings and leaves the layer editable. **Link** keeps the layer synced to the preset, makes layer settings read-only, and propagates preset changes. The embedded snapshot is the fallback when the preset is absent. — **Reversibility:** costly — undo changes profile semantics, UI, and persistence.
- **D-12:** "Extract preset" creates a named profile from the active layer's operation snapshot. It is the path for promoting embedded settings into the profile library. — **Reversibility:** reversible.

#### Migration and Compatibility

- **D-13:** The wire format becomes v2. New documents are written as v2; old builds reject them with the existing format-version error. v1 documents are migrated in memory on load. The file is not rewritten until the user saves; saving writes v2. — **Reversibility:** one-way — v2 is a published file-format contract and undo needs a migration.
- **D-14:** Migration resolves the v1 conflict by effective shape operation, not by `Layer.kind` alone. For each v1 layer, group shapes by effective operation plus `Shape.tool`, create ordered operation layers for each distinct pass, and preserve original layer order and intra-layer shape order. — **Reversibility:** costly — undo needs a reverse migration and loses the richer operation split.
- **D-15:** v1 global `GcodeSettings` seed the default operation settings during migration because v1 has no per-layer settings. Exact field mapping is a planning/research detail. — **Reversibility:** costly — undo requires reworking migration defaults.
- **D-16:** `Layer.kind`, `Shape.style`, and `ShapeProduction.operation` are legacy read-only compatibility fields. They are deserialized for v1 migration, are not production authority, and are not written into v2. — **Reversibility:** costly — undo reintroduces duplicate operation sources.
- **D-17:** Template/guide layers migrate to `Operation::Guide` to preserve non-output semantics. In v2 a template/guide layer may be `None` or `Guide`; both are non-output. The old `template` flag is legacy read-only. — **Reversibility:** costly — undo changes migration and layer semantics.
- **D-18:** A new document starts with one base layer with `operation: None`. It is a workspace container, not an implicit Cut layer. Preferences/templates may assign it an operation. — **Reversibility:** costly — undo changes default-document behaviour and migration expectations.

#### Ordering

- **D-19:** The order of layers with `Some(operation)` is the production sequence. Operation-less layers are not in the sequence. — **Reversibility:** costly — undo changes job ordering and UI semantics.
- **D-20:** Intra-layer shape order is preserved as an ordered `Vec<ShapeId>` (or equivalent). Phase 4 may optimise it; manual reordering comes later. — **Reversibility:** reversible.

### the agent's Discretion

- Exact Rust representation of an operation: data-carrying enum vs `OperationKind` + settings struct. The conceptual model is locked; representation is for research/planning.
- Exact v2 wire field names, shape storage layout, and membership serialization.
- Exact parameter schema per process/strategy and the validation matrix.
- Exact migration mapping from v1 `Operation`/`ToolOp`/`GcodeSettings` to cooked operations.
- Whether `preset_id` is a UUID, slug, or both; and how preset versions are tracked.

### Deferred Ideas (OUT OF SCOPE)

- Layer/Operations UI and "New operation" catalogue — Phase 2.
- Assign/Link UI and profile library management — Phase 2, with model support in Phase 1.
- Per-operation G-code emission and byte parity — Phase 3.
- 2-opt/start-point ordering and manual intra-layer reordering — Phase 4.
- Streaming, recovery, and raster parity — next milestone.
- Exact operation parameter matrix and profile versioning — planning/research.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| MODEL-01 | Document stores shape geometry once, independently of operation layers. | Move `Shape` out of `Layer` into a document-level store; membership by stable `Shape.id: u64`. See *Architecture Patterns → v2 model*. |
| MODEL-02 | An operation layer carries an `Operation`, production settings, colour, visibility, lock, and order. | New `Layer { operation: Option<Operation>, settings/params, color, visible, locked, members }`. Order = `Document.layers: Vec<Layer>` order; see *Standard Stack / Patterns*. |
| MODEL-03 | A shape can belong to zero, one, or many operation layers; same shape in multiple ordered passes. | `Layer.members: Vec<u64>` (N:M); a shape id may appear in many layers. See *Code Examples → membership*. |
| MODEL-04 | Existing `.stencilcraft` v1 documents migrate without geometry or production-data loss. | Version negotiation in `craft-stencilcraft-format` + `migrate_v1` grouping by effective operation + `Shape.tool`; fixtures already exist. See *Migration algorithm* and *Runtime State Inventory*. |
| MODEL-05 | Shapes with no operation membership remain valid, shown "Unassigned", excluded from jobs by default. | Derived `Document::unassigned_shapes()` = ids not in any `Some(operation)` layer's members; never persisted. See *Pitfalls* and *Code Examples*. |
| MODEL-06 | `Operation` covers at least Cut, Engrave, and Score end-to-end; richer variants remain representable. | v2 `Operation` keeps all 9 v1 variants (`Cut, KissCut, DieCut, Draw, Score, Engrave, Perforate, PrintThenCut, Guide`) as cooked presets with settings. See *Standard Stack → Operation*. |
| MODEL-07 | Deleting an operation layer removes memberships, never shape geometry. | Add `Document::remove_layer(index)` dropping only `Layer.members`; geometry stays in the document store. Existing engine `LayerRemove` currently does `doc.layers.remove(index)` (drops embedded shapes in v1). See *Common Pitfalls*. |
</phase_requirements>

## Project Constraints (from AGENTS.md)

Two target repositories carry agent rules that bind this phase's plans. `refs/craft-libs/AGENTS.md` governs the document/format crates; `worktrees/stencilcraft-planning/AGENTS.md` governs the consumer app. Actionable directives to honour:

**craft-libs (document/format live here):**
- Rust **1.95+**, **edition 2024**, workspace resolver 3; `unsafe_code = "forbid"` workspace-wide. [VERIFIED: refs/craft-libs/Cargo.toml:8-10; refs/craft-libs/AGENTS.md]
- A new or changed crate's `Cargo.toml` includes `[lints] workspace = true`. Checks for changed crates: `cargo fmt --check`, `cargo clippy -p <crate> --all-targets -- -D warnings`, `cargo test -p <crate>`. [CITED: refs/craft-libs/AGENTS.md "This repository"]
- One PR covers one issue and changes nothing the fix does not need; moving code and changing behaviour are separate PRs. [CITED: refs/craft-libs/AGENTS.md]
- Non-test code MUST NOT panic; no `allow`/lint-config edits to silence lints; `unsafe` forbidden here. [CITED: refs/craft-libs/AGENTS.md]
- Integration tests SHOULD be one binary per crate at `tests/it/main.rs`. [CITED: refs/craft-libs/AGENTS.md rule 30]
- A test SHOULD guard a behaviour a plausible change could break; assert relations, not chosen constants. [CITED: refs/craft-libs/AGENTS.md rules 27–28]
- Clean-room: never copy GPL/AGPL source; behaviour from public docs only. [CITED: refs/craft-libs/AGENTS.md]
- Versioning: all crates share one workspace version; a change requiring consumer action (public API/persisted-representation change) is a **minor** bump (`0.1.x → 0.2.0`) with migration notes. This phase is such a change. [CITED: refs/craft-libs/docs/versioning.md]

**StencilCraft (consumers, Phases 2–3 adapt these):**
- Never crash; fallible work returns `Result`/`Option`; clippy bans `unwrap/expect/panic/unreachable/todo/unimplemented` outside tests. [CITED: worktrees/stencilcraft-planning/AGENTS.md]
- Everything is a command (`engine.run`); panels change engine state only through commands. [CITED: worktrees/stencilcraft-planning/AGENTS.md]
- Units are millimetres; convert once at a boundary. [CITED: worktrees/stencilcraft-planning/AGENTS.md]
- Quality gate before commit: `cargo xtask ci`. [CITED: worktrees/stencilcraft-planning/AGENTS.md]

**Repo-root lab note:** `rustfactor/.gitignore` ignores `.planning/`, so `commit_docs: true` will not push research/plans from this lab repo. [VERIFIED: rustfactor/.gitignore]

## Summary

The phase is a **schema evolution + in-memory migration** inside two existing craft-libs crates. The current model is fully readable and small: `craft-stencilcraft-document` owns `Document`, `Layer { kind, shapes: Vec<Shape>, … }`, `Shape { id, path, style, tool, production }`, `Operation` (9 variants) and `ShapeProduction`; `craft-stencilcraft-format` owns the versioned JSON envelope with `FORMAT_VERSION = 1` and a **strict `version == FORMAT_VERSION` check** plus a **top-level field whitelist** (`is_wire_field`). Both are hard blockers for v2 and must change: bump the format to 2, accept versions 1 and 2 on read, dispatch by version, and add the v2 top-level field name(s) to the whitelist or they are silently dropped on save/reload.

The migration is a pure function from a v1-typed parse to the v2 in-memory model: for each v1 layer (in order), compute each shape's **effective pass** (precedence: `Shape.tool` → `ShapeProduction.operation` → `Shape.style` fallback), group shapes by that pass, and emit one ordered v2 operation layer per group, preserving layer order and intra-layer shape order (D-14). Geometry is hoisted once into a document-level store; each v2 layer holds ordered `members` (shape ids). The v1 global `GcodeSettings` that seeds settings live in the **StencilCraft app crate**, not in the document or the file, so the seed must be injected into migration (a `MigrationSeed` parameter); `Document::load_json` uses neutral defaults and the app wires `GcodeSettings` in Phase 3.

**Primary recommendation:** Implement v2 as: `Document { shapes: Vec<Shape>, layers: Vec<Layer>, … }` with `Layer { operation: Option<Operation>, members: Vec<u64>, color, visible, locked, name }`; keep **version-specific wire structs** (v1 read-only types + v2 types) so legacy fields are never written; make `craft-stencilcraft-format` negotiate versions 1..=2 and own both field-name sets; add `Document::remove_layer` and `Document::unassigned_shapes`; land it in craft-libs as a `0.2.0`-class change with a `tests/it/main.rs` migration/round-trip suite. Consumer adaptation (gcode/layout/engine/templates) is a **separate StencilCraft PR in Phase 2/3**, not Phase 1.

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| v1/v2 envelope, header, version negotiation, top-level field ownership | `craft-stencilcraft-format` (library) | — | It already owns `FORMAT_VERSION`, `HEADER_KEY`, `DocumentFields`, `is_wire_field`; extending it keeps one wire-schema authority. |
| v2 document model (`Document`, `Layer`, `Shape`, `Operation`) | `craft-stencilcraft-document` (library) | — | The domain model and validation live here; consumers re-export it. |
| v1→v2 migration function | `craft-stencilcraft-document` (library) | format crate supplies detected version | Pure transform, unit-testable with fixtures, no app dependency. |
| Migration settings seed (`GcodeSettings`) | StencilCraft app (`crates/gcode`) → injected into document migration | — | `GcodeSettings` lives in the app crate (`worktrees/.../crates/gcode/src/lib.rs`), not the document or the file; D-15 requires the app to supply it. Phase 3 wires it. |
| Unassigned derivation, layer removal | `craft-stencilcraft-document` (library) | engine (Phase 2 UI) | Derived query and membership removal are model concerns; the engine command wraps them later. |
| Consumer reads of `Layer.kind` / `Shape.style` / `Shape::operation()` | StencilCraft `gcode`, `layout`, `engine`, `doc/templates` (Phase 2/3) | — | They must move to layer operation; out of Phase 1 scope but must be tracked as a rev-bump dependency. |
## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `serde` / `serde_json` | workspace (`serde = "1"`, `serde_json = "1"`) | v1/v2 wire structs, `skip_serializing_if`, version dispatch | Already the only serialization path; the model derives `Serialize/Deserialize`. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/Cargo.toml] |
| `kurbo` (features `serde`) | workspace | `BezPath` geometry, bounds | `Shape.path: BezPath` serializes today; do not change geometry handling. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:13,142] |
| `thiserror` | workspace | `DocError` variants | Extend `DocError`; do not invent a new error type. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:23-45] |
| `craft-atomic` | workspace path dep | atomic file writes | `save_to` already uses it; keep. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:583] |
| `craft-stencilcraft-format` | workspace path dep | versioned envelope + wire field ownership | The header/version/field-name authority; extend it for v2. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:414-426] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `std::collections::BTreeSet` / `BTreeMap` | std | set of ids, deterministic ordering in validation/pruning | For "unassigned"/membership set maths and dangling-id pruning (mirror `pruned_groups`). [VERIFIED: refs/craft-libs/crates/stencilcraft-document/src/canvas.rs:425-433] |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Version-specific v1/v2 wire structs | One `Layer` struct carrying both `kind`+`shapes` (legacy) and `operation`+`members` | Single type avoids duplication but makes "legacy not written into v2" fragile and risks consumers reading an always-empty `layer.shapes` in v2. Rejected. |
| `Document.shapes: Vec<Shape>` | `BTreeMap<u64, Shape>` or a `ShapeStore` newtype | `BTreeMap` enforces unique ids and gives O(log n) lookup with id-order iteration; `Vec` keeps current ergonomics and order but needs a uniqueness check in `validate()`. See Open Question Q6 — recommend `Vec<Shape>` + uniqueness validation for Phase 1 (YAGNI on the index). |
| Data-carrying `Operation` enum | `OperationKind` + `OperationSettings` struct | Both satisfy D-06 discretion. Enum keeps all 9 v1 variants representable (MODEL-06) and makes invalid states hard; struct eases shared settings. See *Standard Stack → Operation*. |

**Installation / version verification:** No new dependencies. This phase only edits existing crates in `refs/craft-libs` (edition 2024, Rust 1.95+, workspace version `0.1.0` → bump to `0.2.0` at release). Verified crate deps: `craft-atomic`, `craft-stencilcraft-format`, `craft-svg`, `kurbo`, `serde`, `serde_json`, `thiserror`. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/Cargo.toml; refs/craft-libs/Cargo.toml:7 version "0.1.0"]

### Operation (v2) — recommended representation

The locked concept is process/tool × strategy × params (D-06); the Rust shape is discretion. Recommend a **data-carrying enum of cooked presets** that still preserves every v1 variant's wire identity, with a shared settings struct:

```rust
// craft-stencilcraft-document (new v2 module), kebab-case wire tags preserved from v1
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "kebab-case")]
pub enum Operation {
    Cut(LaserParams),
    Engrave(LaserParams),
    Score(ScoreParams),
    KissCut(LaserParams),
    DieCut(LaserParams),
    Draw(PenParams),
    Perforate(LaserParams),
    PrintThenCut(PrintParams),
    Guide, // non-output marker (D-17)
}
```
Process/strategy remain derivable (`impl Operation { fn process(&self)->Process; fn strategy(&self)->Strategy }`) so the two-axis model is honoured without exposing raw axes to users (D-06/D-07). The settings struct carries the D-09 snapshot: `power`, `speed_mm_per_min`, `passes`, `focus_mm`, `kerf_mm`, `tabs`, plus strategy-specific params. **All numeric fields must pass `check_finite_mm`-style validation** (see Security). Exact parameter matrix is discretion — keep it minimal and extensible; do not try to model Phase 3 emission here.

## Package Legitimacy Audit

No external packages are installed by this phase. All work uses crates already in the `craft-libs` workspace (`serde`, `serde_json`, `kurbo`, `thiserror`, `craft-atomic`, `craft-stencilcraft-format`). Per the legitimacy gate: **nothing to audit** — no new registry dependencies, no `postinstall` surface (Rust). If a plan is tempted to add a UUID crate for `preset_id`, prefer an in-repo string/slug (see Open Question Q4) to avoid a new dependency.

| Package | Registry | Verdict | Disposition |
|---------|----------|---------|-------------|
| *(none added)* | — | — | No change to dependency graph |

## Architecture Patterns

### System Architecture Diagram

```
  .stencilcraft JSON (on disk, untrusted)
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│ craft-stencilcraft-format                                    │
│  RawWire visitor (keeps only whitelisted top-level fields)   │
│  → read header "stencilcraft": { name, version }             │
│  → version ∈ {1,2}? ──no──▶ Error::Version (old builds reject v2) │
│  → yes: return (version, typed fields)                       │
└─────────────────────────────────────────────────────────────┘
      │ version == 1                         │ version == 2
      ▼                                       ▼
┌───────────────────────────┐        ┌───────────────────────────┐
│ v1 wire structs (READONLY) │        │ v2 wire structs            │
│ Layer{kind, shapes:Vec<Sh>}│        │ Document{shapes, layers}   │
└───────────┬───────────────┘        │ Layer{operation, members}  │
            │ migrate_v1(seed)        └────────────┬──────────────┘
            ▼                                      │
┌──────────────────────────────────────────────────▼──────────────┐
│ craft-stencilcraft-document :: Document (always v2 in memory)    │
│  shapes: Vec<Shape>  ← geometry stored ONCE                      │
│  layers: Vec<Layer>  ← ordered passes; members: Vec<ShapeId>     │
│  derived: unassigned_shapes(), production_sequence()             │
│  validate() → save_json() writes version 2 header                │
└──────────────────────────────────────────────────────────────────┘
```

Primary use cases trace cleanly: **v1 file** → header/version detect → v1 parse → migrate → v2 model → save v2. **v2 file** → detect → v2 parse → validate → model (no migration). **new doc** → `Document::default()` = one base layer `operation: None` (D-18).

### Recommended Project Structure (inside craft-libs)

```
crates/
├── stencilcraft-format/
│   └── src/lib.rs          # FORMAT_VERSION=2, accept 1..=2, v2 field names in whitelist,
│                           #   expose detected version + v1/v2 wire types
└── stencilcraft-document/
    ├── src/lib.rs          # Document { shapes, layers, … }, clean v2 Layer, save/load dispatch
    ├── src/production.rs   # Operation (v2 cooked presets) + settings; legacy v1 op read types
    ├── src/migrate.rs      # NEW: migrate_v1(v1, seed) -> Document; pure, unit-tested
    ├── src/canvas.rs       # update shapes()/page_document/thumbnail/group pruning for v2
    └── tests/
        ├── fixtures/*.stencilcraft   # existing v1 fixtures drive migration tests
        └── it/main.rs                # NEW: v1 migration, v2 round-trip, N:M, unassigned, delete-layer
```

### Pattern 1: Versioned envelope dispatch (format crate)
**What:** The format crate already parses the header, checks name, and rejects `version != FORMAT_VERSION`. For v2, it must accept `1..=2` and tell the caller which version was found.
**When to use:** Every `load_json` path; always.
**Example:**
```rust
// Current strict check (must change): refs/craft-libs/.../stencilcraft-format/src/lib.rs:183-185
if version != FORMAT_VERSION {
    return Err(Error::Version { found: version, supported: FORMAT_VERSION });
}
// Target: accept supported range and surface the detected version to the document crate.
pub const SUPPORTED_VERSIONS: &[u64] = &[1, 2]; // FORMAT_VERSION = 2 is the one written
```
Add `"shapes"` (and any v2-only top-level name) to `is_wire_field`, or the field is skipped during parse and then missing on save. [VERIFIED: refs/craft-libs/crates/stencilcraft-format/src/lib.rs:312-329]

### Pattern 2: Pure migration function
**What:** `fn migrate_v1(v1: V1Document, seed: &MigrationSeed) -> Document`, no I/O, deterministic. Grouping key = effective pass (see below). Emits layers in original order; each layer's `members` in original shape order; all shapes into `Document.shapes` once.
**When to use:** Only on `version == 1` load.
**Example (skeleton):**
```rust
// effective pass precedence (D-14): Shape.tool > ShapeProduction.operation > Shape.style
fn effective_pass(shape: &V1Shape) -> PassKey { /* ToolOp | Operation | CutStyle fallback */ }

for v1_layer in v1.layers {                 // preserve original order (D-14)
    let mut groups: IndexMap<PassKey, Vec<u64>> = IndexMap::new();
    for shape in v1_layer.shapes {          // preserve intra-layer order (D-20)
        hoist_shape_once(&mut doc.shapes, &shape); // geometry stored once (D-02)
        groups.entry(effective_pass(&shape)).or_default().push(shape.id);
    }
    for (pass, members) in groups {         // one ordered operation layer per distinct pass
        doc.layers.push(Layer { operation: Some(pass.into_operation(seed)), members, .. });
    }
}
```
`IndexMap` is **not** in the dependency set; if it is unavailable, use `Vec<(PassKey, Vec<u64>)>` with a linear `find` (small layer sizes) — do not add `indexmap` for this.

### Pattern 3: Derived "Unassigned" and ordered production sequence
**What:** `unassigned_shapes()` = `doc.shapes` ids not present in any layer whose `operation.is_some()`. `production_sequence()` = `doc.layers.iter().filter(|l| l.operation.is_some())` in vector order.
**When to use:** Job/layout consumers (Phase 3) and the UI group (Phase 2); never persisted (D-04, Out of Scope).
**Example:**
```rust
pub fn unassigned_shapes(&self) -> impl Iterator<Item = &Shape> {
    let assigned: BTreeSet<u64> = self.layers.iter()
        .filter(|l| l.operation.is_some())
        .flat_map(|l| l.members.iter().copied()).collect();
    self.shapes.iter().filter(move |s| !assigned.contains(&s.id))
}
```

### Anti-Patterns to Avoid
- **Keeping `Layer.shapes` alongside a document store:** duplicates geometry (violates MODEL-01) and desyncs edits; remove `Layer.shapes` from the v2 type.
- **One struct with all legacy + v2 fields:** silently writes legacy fields or produces always-empty `layer.shapes`; use version-specific wire types.
- **Grouping by `Layer.kind` alone:** loses per-shape `tool`/`production.operation`; D-14 forbids.
- **Persisting "Unassigned":** Out of Scope; derive it.
- **`HashSet`/`HashMap` for members order:** loses intra-layer order (D-20); use `Vec<u64>`.
- **Cascading shape deletion on layer removal:** drops geometry (MODEL-07); remove memberships only.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Versioned JSON envelope / header / duplicate-key / nesting-bomb handling | Custom JSON scanner | `craft-stencilcraft-format`'s `RawWire` visitor + `File` | It already bounds parsing, skips unknown fields, handles dup-key precedence, and yields calm `Error`. [VERIFIED: refs/craft-libs/crates/stencilcraft-format/src/lib.rs:210-361] |
| Atomic file writes | Direct `fs::write` | `craft_atomic::atomic_write` | Existing `save_to` contract. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:583] |
| Bounds / path handling | Bespoke geometry | `kurbo` + `geom::bounds_of_shape` | Already used by `Layer::bounds`/`Shape::bounds`. [VERIFIED: lib.rs:123-134,339-343] |
| Dangling-id cleanup | New ad-hoc loop | Mirror `Document::pruned_groups` | Proven pattern: members that no longer exist are ignored and dropped on save. [VERIFIED: canvas.rs:425-443] |
| Error taxonomy | New error enum | Extend `DocError` | Consumers match on it; keep messages calm/printable. [VERIFIED: lib.rs:23-45] |
| Finite-mm checks | Inline `is_finite` scattered | `geom::check_finite_mm` | Shared with board/kerf/image validation. [VERIFIED: lib.rs:11,500-512] |

**Key insight:** The hard parts (untrusted parsing, atomic I/O, geometry bounds) are already solved in these two crates. Phase 1 is a schema transform, not new infrastructure — every new helper should mirror an existing one.

## Runtime State Inventory

> This is a migration-and-refactor phase (renames/relocates persisted shape ownership and adds a format version), so the inventory is mandatory.

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| **Stored data** | v1 `.stencilcraft` files on users' disks and in the local library. Committed copies: `crates/stencilcraft-document/tests/fixtures/{starter,poster,stencil-frame}.stencilcraft` and `examples/{starter,poster,stencil-frame}.stencilcraft` (all `"version": 1`, layers carry `kind` + inline `shapes`, shapes carry `style`/`tool`/`production`). [VERIFIED: fixtures read this session] | In-memory migration only; files are **not** rewritten until the user saves (D-13). Add the v1 fixtures to the migration test corpus. No bulk data migration script. |
| **Live service config** | None — StencilCraft has no server/DB; project/kanban state is not stored in the document. Templates are **built in code** (`crates/doc/src/templates.rs`), not persisted. [VERIFIED: worktrees/.../crates/doc/src/templates.rs:345-461] | None. |
| **OS-registered state** | None — no task scheduler/pm2/launchd/systemd entries reference the model. | None — verified by absence of any such integration in the repos. |
| **Secrets / env vars** | None — no secret key or env var names embed `Layer`/`Shape`/`Operation`; the document holds no credentials. | None. |
| **Build artifacts / installed packages** | The StencilCraft app pins `craft-stencilcraft-document`/`-format` at git rev `75c70fa6a…` (workspace `Cargo.toml:31-32`), so it keeps compiling against v1 until that rev is bumped. `[patch.crates-io]` exists only for hayro/egui/winit, **not** for the stencilcraft crates — the local-dev `[patch]` described in PROJECT.md is not wired yet. Cargo git checkouts of the old rev are cached under `~/.cargo`. [VERIFIED: worktrees/.../Cargo.toml:31-32,106-126] | A Phase-1 setup task must create a local craft-libs dev branch/worktree and add a `[patch]` (or a branch rev) so StencilCraft can compile against v2. Bump the rev in the Phase 2/3 consumer PR. Bump `craft-libs` workspace version to `0.2.0` and add a CHANGELOG entry at release. |

## Common Pitfalls

### Pitfall 1: The format reader rejects every v1 file the moment `FORMAT_VERSION` becomes 2
**What goes wrong:** Loading an old `.stencilcraft` returns `Error::Version { found: 1, supported: 2 }`; the app cannot open existing work.
**Why it happens:** `from_typed_wire`/`from_wire` require `version == FORMAT_VERSION` exactly. [VERIFIED: refs/craft-libs/crates/stencilcraft-format/src/lib.rs:183-185]
**How to avoid:** Introduce a supported-version range/allow-list and a versioned read path that returns the detected version; dispatch to v1 parse→migrate or v2 parse.
**Warning signs:** Migration tests using the existing v1 fixtures fail at parse, before migration runs.

### Pitfall 2: New v2 top-level fields are silently dropped by the `is_wire_field` whitelist
**What goes wrong:** A saved v2 document reloads with an empty shapes list (or missing membership data) because the reader skipped the field.
**Why it happens:** `RawWire` keeps only names matched by `is_wire_field`; the current list ends at `meta` and does not include `shapes`. [VERIFIED: refs/craft-libs/crates/stencilcraft-format/src/lib.rs:312-329]
**How to avoid:** Register every v2 top-level field name; keep `shapes` and `layers` both allowed. Add a round-trip test asserting `shapes` survives save→load.
**Warning signs:** `save_json` output contains `shapes`, but `load_json` yields zero shapes.

### Pitfall 3: Geometry duplicated or lost across the v1→v2 move
**What goes wrong:** Shapes exist both in `Layer.shapes` and `Document.shapes` (edited twice, MODEL-01 broken), or layer removal deletes shapes (MODEL-07 broken).
**Why it happens:** v1 stores shapes inside layers and the engine's `LayerRemove` does `self.doc.layers.remove(index)` — dropping embedded shapes. [VERIFIED: worktrees/.../crates/engine/src/lib.rs:1315-1327]
**How to avoid:** Remove `Layer.shapes` from the v2 type entirely; hoist each shape once; implement `Document::remove_layer` to drop only memberships; keep ids stable via `next_shape_id`. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:387-389,467-479]
**Warning signs:** A shape id appears in two `Document.shapes` entries; deleting a layer reduces `shape_count`.

### Pitfall 4: Intra-layer order collapses to set order
**What goes wrong:** Engrave-before-cut or interior-before-outline ordering changes between save and reload.
**Why it happens:** Using `HashSet`/`HashMap`/`BTreeSet` for `members` reorders; D-20 needs an ordered list.
**How to avoid:** `members: Vec<u64>`; preserve append order from the source. [CITED: 01-CONTEXT.md D-20]
**Warning signs:** Round-trip test comparing `members` vectors fails only on order.

### Pitfall 5: Migration seeds settings from the wrong place (or not at all)
**What goes wrong:** Migrated layers get no/incorrect power/speed/passes, so a re-sent job differs; or the document crate gains an app dependency.
**Why it happens:** `GcodeSettings` lives in the StencilCraft `gcode` crate, not the document crate or the file. [VERIFIED: worktrees/.../crates/gcode/src/lib.rs:24-30]
**How to avoid:** Define a neutral `MigrationSeed`/`OperationDefaults` in the document crate; `load_json` uses defaults, `load_json_with_seed` takes the app's mapped `GcodeSettings`. Wire the app in Phase 3. [CITED: 01-CONTEXT.md D-15]
**Warning signs:** A `craft-stencilcraft-document` patch tries to `use stencilcraft_gcode::GcodeSettings` (layer violation).

### Pitfall 6: `Document::default()` keeps producing an implicit Cut layer
**What goes wrong:** New documents start with a `Cut` layer, contradicting D-18 and the "no implicit Cut" rule.
**Why it happens:** `Document::default` sets `layers: vec![Layer::new("cut", CutStyle::Cut)]`. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:433]
**How to avoid:** Default to one base layer with `operation: None`, and a `Layer::new`-style constructor that takes an operation option.
**Warning signs:** Tests assert `layers[0].kind == CutStyle::Cut` — they must move to operation assertions.

### Pitfall 7: Dangling memberships and duplicate ids after edits
**What goes wrong:** `members` references a deleted shape id, or `Document.shapes` holds two shapes with the same id; validation or output becomes inconsistent.
**Why it happens:** No referential check in v2 by default.
**How to avoid:** In `validate()`, assert unique shape ids and that every member id exists (or prune like `pruned_groups`); on load, prune inactive ids and drop unknown members. [VERIFIED: canvas.rs:425-443]
**Warning signs:** A `get_shape(id)` returns `None` for an id present in a layer's members.

### Pitfall 8: Non-finite settings reach `save_json`
**What goes wrong:** NaN/Infinity power/speed/focus/kerf serializes to invalid JSON or fails to re-parse.
**Why it happens:** New settings floats are not routed through the existing finite checks. [VERIFIED: lib.rs:498-514]
**How to avoid:** Validate every new numeric field with `check_finite_mm`/explicit finite checks in `Document::validate`.

## Code Examples

Verified patterns from the current sources (adapt, don't copy blindly):

### v2 document and layer (target shape)
```rust
// craft-stencilcraft-document — v2 in-memory model (recommended)
pub struct Document {
    pub board: Board,
    pub kerf: KerfSettings,
    pub shapes: Vec<Shape>,          // geometry stored ONCE (MODEL-01, D-02)
    pub layers: Vec<Layer>,          // ordered passes + containers (D-19)
    pub next_shape_id: u64,          // existing id allocator
    pub images: Vec<RasterImage>,    // unchanged
    // board_setup, pages, guides, grid, snap, groups, meta unchanged
}

pub struct Layer {
    pub name: String,
    pub operation: Option<Operation>, // None = container, not a pass (D-01, D-18)
    pub visible: bool,
    pub locked: bool,
    pub color: Option<String>,
    pub members: Vec<u64>,            // ordered ShapeId, N:M (D-20, MODEL-03)
}
```
Existing field names/behaviour to preserve: `Layer.color` is `Option<String>` `#rrggbb`; `Layer.locked`/`Layer.visible`; `Document.next_shape_id` maintained by mutators. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:89-104,383-410]

### Effective-pass precedence for migration (D-14)
```rust
// Precedence: Shape.tool (ToolOp) > ShapeProduction.operation > Shape.style fallback.
// ToolOp is None on plain paths; when present it overrides (v1 says ToolOp lives on engrave layers).
// Shape.style fallback is what Shape::operation() already does:
//   CutStyle::Cut => Operation::Cut, CutStyle::Engrave => Operation::Engrave
// [VERIFIED: refs/craft-libs/.../production.rs:200-205; lib.rs:176-205]
```
Map v1 `ToolOp` → v2 `Operation`: `Score{tip,passes}` → Score; `Emboss{…}` → Engrave/emboss; `PowerEngrave` → Engrave; `HeatPen` → Print/foil; `Punch{diameter_mm}` → Punch. Carry `tip`, `passes`, `deboss`, `diameter_mm` into the layer's settings snapshot. [VERIFIED: ToolOp variants at lib.rs:181-205]

### Unassigned derivation + layer deletion (MODEL-05, MODEL-07)
```rust
impl Document {
    /// Derived; never persisted (D-04). Shapes in only operation-less layers are unassigned.
    pub fn unassigned_shapes(&self) -> impl Iterator<Item = &Shape> { /* ids not in any Some(op) layer */ }

    /// Removes the layer and its memberships; geometry stays (MODEL-07). Keep ≥1 layer? (see Q7)
    pub fn remove_layer(&mut self, index: usize) -> Result<(), DocError> {
        // drop the Layer (with its members) only — never touch self.shapes
    }
}
```

### save/load dispatch (format crate + document crate)
```rust
// load_json_named (document crate): detect version, then parse/dispatch.
let (version, raw) = FormatFile::read_versioned(json)?; // new API; accepts 1..=2
match version {
    1 => migrate_v1(parse_v1(raw)?, &MigrationSeed::default()),
    2 => parse_v2(raw)?,
    _ => unreachable in practice (format crate already rejected),
}
// save_json: build v2 wire, write header version 2.
```
`Document::save_json` today builds `NativeFileFields` and calls `File::to_json_pretty`. [VERIFIED: refs/craft-libs/crates/stencilcraft-document/src/lib.rs:519-538]

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Shapes owned by `Layer` (`Layer.shapes: Vec<Shape>`) | Geometry stored once; layers reference by id; N:M membership | This phase (v2) | Enables engrave-then-cut on one shape; changes every consumer's iteration. |
| Operation as per-shape `ShapeProduction.operation` + `Shape.style` fallback | Operation on the layer; optional marker; `ToolOp` folded in | This phase (v2) | One production authority; legacy fields read-only for migration. |
| `Layer.kind: CutStyle { Cut, Engrave }` | `Layer.operation: Option<Operation>` (9 cooked variants + settings) | This phase (v2) | Cut/Engrave/Score + richer variants end-to-end. |
| Strict `version == FORMAT_VERSION` load | Accept `1..=2`, migrate v1 in memory, write v2 | This phase | Old builds reject v2 with the existing version error; v1 keeps loading. |

**Deprecated/outdated (legacy, read-only, do not write into v2):**
- `Layer.kind: CutStyle` — v1 only (D-16).
- `Shape.style: CutStyle` — v1 only (D-16).
- `Shape.tool: Option<ToolOp>` — migrates into the layer operation (D-08).
- `ShapeProduction.operation: Option<Operation>` — migrates into the layer operation (D-16).
- v1 `template: bool` flag — becomes `Operation::Guide` or `None` (D-17).

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Phase 1 lands only in `craft-libs` (document + format) and its own tests; StencilCraft consumer crates are adapted in Phase 2/3 with a rev bump. | Summary / Runtime State Inventory | If Phase 1 must keep the StencilCraft workspace compiling, the plan needs additional consumer-adaptation tasks and a `[patch]`; scope grows substantially. **Confirm with user (Q1).** |
| A2 | The v2 `Operation` can be a data-carrying enum of the 9 v1 variants + settings, with process/strategy as derived accessors. | Standard Stack → Operation | If the user prefers explicit process+strategy fields, the schema and migration grouping key change. Discretion area (D-06). |
| A3 | `GcodeSettings` (`feed_mm_per_min`, `power`, `engrave_power`, `passes`, `ramp_power`, `mode`) is the full seed for migrated settings. | Pitfall 5 | If Phase 3 adds more emission settings, the snapshot must grow before release; a second migration risk. |
| A4 | The `craft-libs` workspace version bumps to `0.2.0` for this change. | Project Constraints | If maintainers want a patch release, the persisted-representation change would violate the documented policy. |
| A5 | `Document.shapes: Vec<Shape>` (with uniqueness validation) is adequate; no indexed store is needed now. | Standard Stack → Alternatives | Large documents may want O(1) lookup; can add an id→index map later without a format change. |
| A6 | No new external dependency is needed (including for `preset_id` / ordering). | Standard Stack / Package Audit | If a UUID or ordered-map crate is added, it re-triggers the legitimacy gate and licence check. |
| A7 | The local build can fetch/compile the two crates (registry cache present; network presumed). | Environment Availability | If offline and deps uncached, first build fails; vendored deps or a warmed cache would be required. |

## Open Questions (RESOLVED)

1. **Where does Phase 1 code physically land? — RESOLVED.**
   - What we know: `refs/craft-libs` is a pristine clone on `main` (commit `249a41b`, workspace `0.1.0`); the StencilCraft dev worktree is `worktrees/stencilcraft-planning` (branch `planning`) and pins the document crates at rev `75c70fa6a…`; there is no local craft-libs branch and no `[patch]` for the stencilcraft crates. [VERIFIED: git + Cargo.toml inspection]
   - Answer: Phase 1 is a craft-libs-only edit plus tests; the plans run from the workspace root with `--manifest-path refs/craft-libs/Cargo.toml` and do not edit `rustfactor/` or `worktrees/`. The StencilCraft `[patch]`/branch wiring is Phase 2 dev setup, not a Phase 1 task.

2. **Must Phase 1 keep the StencilCraft workspace compiling? — RESOLVED.**
   - What we know: changing `Document`/`Layer`/`Shape` breaks `gcode` (`layer.kind`), `layout` (`Shape::operation()`), `engine` (`shape.style = l.kind`, `layer.shapes`), and `doc/templates` (`Layer::new`, `doc.layers[i].shapes`).
   - Answer: No. Consumer adaptation is Phase 2/3 work in the StencilCraft repo; the 0.2.0 CHANGELOG is the release-coordination note for the temporary rev gap.

3. **Exact `Operation` representation (enum vs kind+settings)? — RESOLVED.**
   - Answer: data-carrying enum tagged by `kind` (kebab-case), with `OperationSettings` carrying power, speed, passes, focus, kerf, tabs, tip, deboss, and punch diameter. `process() -> Option<Process>` (`None` for Guide) and `strategy() -> Strategy` are derived accessors (D-06/D-07).

4. **`preset_id` form? — RESOLVED.**
   - Answer: `preset_id: Option<String>` as a stable slug, plus `preset_linked: bool` on the layer. `preset_id` is provenance only (D-10); `preset_linked` distinguishes Assign from Link (D-11) without a UUID dependency or preset-version machinery in Phase 1.

5. **Assign vs Link model support in Phase 1 — RESOLVED.**
   - Answer: `preset_id` + `preset_linked` on the embedded layer snapshot is the full Phase 1 model support. Propagation and read-only enforcement are Phase 2 UI work.

6. **Shape store: `Vec<Shape>` vs `BTreeMap<u64, Shape>` — RESOLVED.**
   - Answer: `Vec<Shape>` plus unique-id validation in `validate_model`. Revisit only if lookup performance becomes measurable.

7. **Minimum layer invariant after delete — RESOLVED.**
   - Answer: the model allows zero layers. `Document::remove_layer` removes any layer, including the last, and never touches `Document.shapes`; the engine keeps its "document must keep at least one layer" rule (`worktrees/.../crates/engine/src/lib.rs:1321-1325`) in Phase 2. The 01-03 plan tests both delete-keeps-geometry and delete-last-layer.

8. **Does migration ever produce a `None` layer? — RESOLVED.**
   - Answer: no. v1 has no operation-less layer and every v1 shape's `style` maps to an operation, so migration always emits operation layers; `operation: None` layers arise only from new/edited v2 documents or the D-18 default document.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|-------------|-----------|---------|----------|
| `cargo` | build/test the crates | ✓ | 1.98.0 | — |
| `rustc` | edition 2024 / MSRV 1.95 | ✓ | 1.98.0 | — |
| Cargo registry cache | offline/parallel builds | ✓ (≈1.4 GB) | — | warm cache / vendor |
| Network to crates.io | first build (target dir absent) | assumed ✓ | — | `--offline` if cache complete |
| `refs/craft-libs` git repo | source of truth | ✓ | main @ `249a41b`, v0.1.0 | — |
| `worktrees/stencilcraft-planning` | consumer reference + rev pin | ✓ | branch `planning`, pin `75c70fa6a…` | — |

**Missing dependencies with no fallback:** None identified.
**Missing dependencies with fallback:** Network (fallback `--offline` with the warm cache); first build has no `target/` yet, so compile time is expected.

## Validation Architecture

> Included because `workflow.nyquist_validation` is `true` in `.planning/config.json`.

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Rust built-in `#[test]` / `cargo test` (no external test crate) |
| Config file | none — inline `#[cfg(test)]` modules (e.g. `refs/craft-libs/crates/stencilcraft-document/src/lib.rs:591-846`) plus `tests/fixtures/` |
| Quick run command | `cargo test -p craft-stencilcraft-document -p craft-stencilcraft-format --locked` |
| Full suite command | `cargo test -p craft-stencilcraft-document -p craft-stencilcraft-format --all-targets --locked` then `cargo clippy -p craft-stencilcraft-document -p craft-stencilcraft-format --all-targets -- -D warnings` and `cargo fmt --check` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| MODEL-01 | Shape geometry stored once; save/reload preserves it | unit/round-trip | `cargo test -p craft-stencilcraft-document round_trip --locked` | ❌ Wave 0 (v2 round-trip) |
| MODEL-02 | Layer carries operation/settings/colour/visible/lock/order and round-trips | unit | `cargo test -p craft-stencilcraft-document layer_round_trip --locked` | ❌ Wave 0 |
| MODEL-03 | One shape in two layers saves/reloads with two memberships | unit | `cargo test -p craft-stencilcraft-document shape_in_two_layers --locked` | ❌ Wave 0 |
| MODEL-04 | v1 fixtures migrate with geometry + production preserved | integration | `cargo test -p craft-stencilcraft-document migrate_v1 --locked` | ❌ Wave 0 (fixtures exist) |
| MODEL-05 | Zero-membership shape is valid and reported unassigned | unit | `cargo test -p craft-stencilcraft-document unassigned --locked` | ❌ Wave 0 |
| MODEL-06 | Cut/Engrave/Score + richer variants round-trip | unit | `cargo test -p craft-stencilcraft-document operation_round_trip --locked` | ❌ Wave 0 |
| MODEL-07 | Remove layer drops memberships, keeps geometry | unit | `cargo test -p craft-stencilcraft-document remove_layer --locked` | ❌ Wave 0 |
| (format) | v2 detected, v1 accepted, old-build rejection preserved | unit | `cargo test -p craft-stencilcraft-format --locked` | ✅ existing tests adapt |

### Sampling Rate
- **Per task commit:** `cargo test -p craft-stencilcraft-document -p craft-stencilcraft-format --locked`
- **Per wave merge:** the two-crate test + `cargo clippy … -D warnings` + `cargo fmt --check`
- **Phase gate:** full two-crate test suite green; migration fixtures pass; no `unwrap/panic` added outside tests.

### Wave 0 Gaps
- [ ] `crates/stencilcraft-document/tests/it/main.rs` — integration binary covering MODEL-01..07 (craft-libs rule 30).
- [ ] Migration tests over `tests/fixtures/{starter,poster,stencil-frame}.stencilcraft` (MODEL-04).
- [ ] v2 round-trip test with a shape in two ordered layers (MODEL-01/03).
- [ ] Unassigned-derivation test (MODEL-05).
- [ ] `remove_layer` test (MODEL-07).
- [ ] Operation round-trip test incl. all 9 variants (MODEL-06).
- [ ] Format-crate version-negotiation tests (v1 accepted, v2 written, v3 rejected).
- [ ] No test framework install needed (built-in cargo).

## Security Domain

> `security_enforcement` is `true`, `security_asvs_level` 1.

### Applicable ASVS Categories
| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | no auth surface in a document library |
| V3 Session Management | no | no sessions |
| V4 Access Control | no | no subjects/resources |
| V5 Input Validation | **yes** | `serde` typed parsing + `RawWire` visitor + `DocError` `Result`; finite-mm checks (`check_finite_mm`); bounded image/notes sizes (`MAX_IMAGES`, `MAX_NOTES_BYTES`, `MAX_IMAGE_DATA`). [VERIFIED: refs/craft-libs/crates/stencilcraft-format/src/lib.rs:210-361; .../stencilcraft-document/src/lib.rs:11,346-378,498-514] |
| V6 Cryptography | no | no secrets/crypto; do not hand-roll any |

### Known Threat Patterns for this stack (untrusted `.stencilcraft` JSON)
| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Nesting / size bomb in JSON | Denial of Service | Bounded reader + visitor skips unknown deeply-nested values (`IgnoredAny`). [VERIFIED: format lib.rs:233-248] |
| NaN/Infinity or negative dimensions in new settings | Tampering / DoS | Reject non-finite/negative in `validate()` before save. |
| Duplicate keys / type confusion | Tampering | `serde_json` last-wins + typed decode; covered by the corpus test pattern. [VERIFIED: worktrees/.../crates/doc/tests/corpus.rs] |
| Dangling membership ids / id collisions | Tampering | Validate unique shape ids and membership referential integrity; prune like `pruned_groups`. |
| Panic on malformed input | DoS | No `unwrap/expect/panic` outside tests; return `Result` (craft-libs clippy + AGENTS.md). |

**Security verdict:** No new attack surface beyond the existing document reader. New fields must join the existing finite/bounds validation; migration must remain panic-free on hostile input (extend the fuzz-lite corpus).

## Sources

### Primary (HIGH confidence — read from source this session)
- `refs/craft-libs/crates/stencilcraft-document/src/lib.rs` — `CutStyle` (:48-55), `Layer` (:89-135), `Shape` (:139-161), `ToolOp` (:176-205), `Document` (:381-410), `Default` (:428-445), `shapes()/get_shape/push_shape` (:447-492), `save_json/load_json` (:519-589).
- `refs/craft-libs/crates/stencilcraft-document/src/production.rs` — `Operation` 9 variants (:13-34), `ShapeProduction` (:141-174), `Shape::operation()` fallback (:199-205), `cut_path` (:176-198).
- `refs/craft-libs/crates/stencilcraft-format/src/lib.rs` — `FORMAT_VERSION=1` (:34), `DocumentFields` (:42-93), strict version check (:183-185), `RawWire` + `is_wire_field` (:205-329), `Error` (:345-361).
- `refs/craft-libs/crates/stencilcraft-document/src/canvas.rs` — `pruned_groups` (:425-443), `page_document`, `render_thumbnail` shape/layer usage.
- `refs/craft-libs/crates/stencilcraft-document/src/metadata.rs` — library/recent-files records (not document state).
- `refs/craft-libs/Cargo.toml` — edition 2024, Rust 1.95, workspace version 0.1.0; `docs/versioning.md` — 0.2.0 policy.
- `refs/craft-libs/crates/stencilcraft-document/tests/fixtures/*.stencilcraft` + `worktrees/stencilcraft-planning/examples/*.stencilcraft` — v1 wire shape (`kind`, inline `shapes`, `style`, `version:1`).
- `worktrees/stencilcraft-planning/crates/gcode/src/lib.rs` — `GcodeSettings` (:24-30), `export` reading `layer.kind` (:76-130).
- `worktrees/stencilcraft-planning/crates/engine/src/lib.rs` — `Command::LayerAdd { name, kind: CutStyle }` (:234), `LayerRemove { index }` (:239-240, :1315-1327), style→layer logic (:1050-1080).
- `worktrees/stencilcraft-planning/crates/engine/src/document.rs` — layer commands (:75-91), `ShapesMoveToLayer`/`shape.style = l.kind` (:700-720), rename/lock/template/color (:726-763).
- `worktrees/stencilcraft-planning/crates/doc/src/lib.rs` (re-export) + `templates.rs` — consumer document construction.
- `worktrees/stencilcraft-planning/crates/doc/tests/corpus.rs` — untrusted-input corpus pattern to extend.

### Secondary (MEDIUM confidence)
- `.planning/REQUIREMENTS.md`, `.planning/ROADMAP.md`, `.planning/PROJECT.md`, `.planning/STATE.md` — scope, constraints, decisions.
- `.planning/phases/01-operation-first-document-model/01-CONTEXT.md` — locked decisions D-01..D-20.
- `refs/craft-libs/AGENTS.md`, `worktrees/stencilcraft-planning/AGENTS.md` — project rules.

### Tertiary (LOW confidence / deferred)
- LightBurn Material Library Assign-vs-Link reference (`https://docs.lightburnsoftware.com/2.2/Reference/MaterialLibrary/`) — cited in CONTEXT; relevant to Phase 2 UI, not Phase 1 code. Not fetched/verified this session.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — dependencies and versions read directly from `Cargo.toml`s; no new packages.
- Architecture (current model + migration mechanics): HIGH — every struct/enum/version check read from source with line numbers.
- v2 `Operation` settings schema: MEDIUM — concept locked (D-06/D-09), exact representation is discretion; recommended enum + settings.
- Cross-repo execution wiring: LOW/ASSUMED — where Phase 1 lands and whether consumers must stay green is not yet decided (Open Q1/Q2).
- Pitfalls: HIGH — each is grounded in a specific line of the existing code (version check, field whitelist, `Document::default`, `LayerRemove`).

**Research date:** 2026-11-24
**Valid until:** ~2026-12-24 (stable Rust crates; re-check if craft-libs rev moves or Phase 2 changes the document schema assumptions).