# Walking Skeleton — StencilCraft Operation-First Document Model

**Phase:** 1
**Generated:** 2026-11-24

## Capability Proven End-to-End

A v1 `.stencilcraft` file loads, migrates in memory to geometry-once + ordered operation layers, saves as a v2 file, and reloads byte-equivalent — with every shape's geometry stored once at document level and referenced by the layers it belongs to.

## Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Crate boundary | `craft-stencilcraft-format` owns the envelope; `craft-stencilcraft-document` owns the model | Envelope version negotiation and model shape are separate concerns; the format crate stays domain-agnostic (D-01) |
| Geometry storage | `Document.shapes: Vec<Shape>`; layers hold `members: Vec<u64>` ids | One geometry record per shape, N:M membership; fixes duplicate-geometry drift (D-02) |
| Operation ownership | `Layer.operation: Option<Operation>`; a `Some(op)` layer includes all its members | Production intent lives on the ordered layer list, not per shape (D-01, D-03, D-19) |
| Legacy compatibility | v1 read-only wire types (`wire_v1`) + pure `migrate_v1`; `Shape.style`/`ShapeProduction.operation` never written again | Preserves old files without carrying dead fields into v2 (D-14, D-16) |
| Version negotiation | `FORMAT_VERSION = 2`, `SUPPORTED_VERSIONS = &[1, 2]`, versioned readers return `(version, File<T>)` | Readers must accept v1 for migration while writers emit v2 (D-13) |
| Template layers | v1 `template: true` layers migrate to `Operation::Guide` | Guide is a real operation that participates in membership but produces no output (D-17) |
| Serialization | `serde` JSON, internally-tagged `Operation` (`kind`), kebab-case tags | Matches existing envelope conventions and keeps wire diffs readable |
| Release | workspace version `0.2.0` | 0.x minor signals a breaking wire/API change per `docs/versioning.md` |

## Stack Touched in Phase 1

- [x] Crate scaffold — existing `craft-stencilcraft-format` and `craft-stencilcraft-document` crates, no new dependencies
- [x] Format envelope — versioned read (`SUPPORTED_VERSIONS`), v2 write header, `shapes` wire field
- [x] Data model — `Document.shapes`, `Layer.members`/`Layer.operation`, slim v2 `Shape`
- [x] Migration — v1 fixture files load through `wire_v1` + `migrate_v1` and re-save as v2
- [x] Derived behavior — `unassigned_shapes`, `production_sequence`, `remove_layer`, canvas reads over membership
- [x] Release metadata — `0.2.0` version bump and CHANGELOG migration note

## Out of Scope (Deferred to Later Slices)

- Consumer UI, editor, and undo integration in `stencilcraft`/`rustfactor` (Phase 2+)
- Kerf compensation, cut ordering, and machine backends (other crates)
- New operation kinds beyond what v1 could express plus the lossless `Punch` mapping
- Migrating v2 files back down to v1 (one-way upgrade only)
- Multi-page/board semantics changes beyond what v1 already stored

## Subsequent Slice Plan

Each later phase adds a vertical slice on top of this skeleton without altering its architectural decisions:

- Phase 2: consumer document model migration and editor wiring against the v2 API
- Phase 3: production sequencing and export paths consuming `production_sequence`
- Phase 4: preset linking and material-aware defaults on operation layers
