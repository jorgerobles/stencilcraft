# Roadmap: Document Organization — LaserCraft/StencilCraft

## Overview

This milestone turns StencilCraft's conflated layer/operation/shape model into an
operation-first document: geometry stored once, ordered operation layers as the
production passes, N:M membership, and a derived "Unassigned" group. From that
foundation it builds the Layer/Operations UI, per-operation laser output with
byte parity across desktop/CLI/MCP, and measurable job planning. Work lands in
`craft-libs` (document) and StencilCraft (layout, machine, engine, UI); the
`rustfactor` lab supplies prototypes and conformance vectors only.

Reliable GRBL streaming, recovery, and raster parity are a separate milestone and
are intentionally not phases of this roadmap.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Operation-First Document Model** - Geometry stored once, ordered operation layers, N:M membership, v1 migration
- [x] **Phase 2: Layer/Operations UI and Assignment** - Real Layers menu, operation-required layers, assignment and ordering UX
- [x] **Phase 2.1: Geometry/Operations Separation (INSERTED)** - Import keeps source nesting as groups; operations become first-class entities (v2 revised in place, no v3) (completed 2026-11-25)
- [ ] **Phase 3: Per-Operation Laser Output** - Power/feed/passes per operation, ordered multi-pass shapes, desktop/CLI/MCP byte parity
- [ ] **Phase 4: Job Planning and Laser Features** - 2-opt and start-point ordering with measured travel, tabs/overcut/focus

## Phase Details

### Phase 1: Operation-First Document Model
**Goal**: A persisted document stores each shape's geometry once and expresses ordered production passes through N:M membership in operation layers, with existing v1 documents migrating intact.
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: MODEL-01, MODEL-02, MODEL-03, MODEL-04, MODEL-05, MODEL-06, MODEL-07
**Success Criteria** (what must be TRUE):
  1. A document saves and reloads with each shape's geometry stored once while that same shape appears in two ordered operation layers.
  2. Loading an existing v1 `.stencilcraft` file preserves every shape's geometry and production data and maps its former layers to operation layers.
  3. A shape with zero memberships saves, reloads, and is reported as unassigned while contributing nothing to any job.
  4. Deleting an operation layer removes its memberships and leaves all shape geometry intact and still editable.
  5. Cut, Engrave, and Score each select an operation on a layer end-to-end, and the richer `Operation` variants remain representable and round-trip.
**Plans**: 4 plans
**Wave 1**
- [x] 01-01-PLAN.md — v2 envelope + geometry-once model + v1 migration (tracer)

**Wave 2** *(blocked on Wave 1 completion)*
- [x] 01-02-PLAN.md — integration tests for the v2 round trip and tool-pass migration

**Wave 3** *(blocked on Wave 2 completion)*
- [x] 01-03-PLAN.md — derived queries, layer deletion, pruning, validation, and canvas tests

**Wave 4** *(blocked on Wave 3 completion)*
- [x] 01-04-PLAN.md — hostile-input coverage, full fixture migration, and 0.2.0 release metadata

### Phase 2: Layer/Operations UI and Assignment
**Goal**: Users can build an ordered operation stack in StencilCraft and assign any selection of objects to it without changing geometry.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: UI-01, UI-02, UI-03, UI-04, UI-05, UI-06, UI-07
**Success Criteria** (what must be TRUE):
  1. User can open a Layers/Operations menu and add a layer only by choosing an operation; no layer is ever created implicitly as Cut. *(wording superseded by locked decision D-01: add-layer creates an operation-less container, operation applied afterwards via popup/palette)*
  2. User can add one or many selected objects to a layer and remove them again, and object geometry stays exactly where it was.
  3. User can reorder operation layers and see the resulting production order in the panel.
  4. User can edit a layer's colour, settings, visibility, and lock from the Layers panel and see the change reflected in the canvas.
  5. The Layers panel shows operation, colour, visibility, lock, and order per layer, object rows show their memberships, and an "Unassigned" group lists exactly the zero-membership shapes without being persisted.
**Plans**: 5 plans
**Wave 1**
- [ ] 02-01-PLAN.md — publish v2 model on phase-1-v2-model, re-pin worktree, library-crate migration + compile gate (tracer)

**Wave 2** *(blocked on Wave 1 completion)*
- [ ] 02-02-PLAN.md — engine/ui-egui/apps/MCP migration, engine membership/operation commands, preset catalogue

**Wave 3** *(blocked on Wave 2 completion)*
- [ ] 02-03-PLAN.md — read-only Layers tree, Pass N badges, Unassigned group, layer-colour canvas paint, inspector effective operation

**Wave 4** *(blocked on Wave 3 completion)*
- [ ] 02-04-PLAN.md — operation popup/palette assignment, Send to Layer… additive assign/remove

**Wave 5** *(blocked on Wave 4 completion)*
- [ ] 02-05-PLAN.md — drag-handle reorder with live Pass N, colour picker, visibility/lock/settings editing

**UI hint**: yes

### Phase 2.1: Geometry/Operations Separation (INSERTED)
**Goal**: Imported geometry stays untouched (source `<g>` nesting becomes document groups) and operations become distinct production entities created from a layer, group, or selection — separating geometry navigation from operation assignment.
**Mode:** mvp
**Depends on**: Phase 2 (corrects the "layer = operation" framing)
**Requirements**: GEO-01, GEO-02, GEO-03, GEO-04, GEO-05, GEO-06, GEO-07, GEO-08
**Context**: `02.1-CONTEXT.md`
**Success Criteria** (what must be TRUE):
  1. A nested SVG imports with identical group/shape nesting and zero synthesized operation layers; a preference restores the legacy semantics.
  2. The geometry panel shows the group/shape tree with enter-group/deep-select and group/ungroup.
  3. An operation is created from a layer/group/selection via the hammer, with id, icon, number, colour, and profile; a shape can belong to several operations.
  4. Geometry rows show operation tags with overflow, and assigned geometry can be filtered by operation.
  5. Bulk creation (by colour / all selected / each selected) confirms above the preference threshold (default 10).
  6. v1 documents still migrate to the revised model with no geometry or production-data loss; the unpublished v2 format is revised in place.
**Plans**: 3/3 plans complete
**Wave 1**
- [x] 02.1-01-PLAN.md — import ADR + geometry-preserving importer + legacy preference + tests

**Wave 2** *(blocked on Wave 1 completion)*
- [x] 02.1-02-PLAN.md — model: `Document.operations` first-class, `Layer` geometry-only, engine/CLI/MCP swap (v2 revised in place; v1 migration one hop)

**Wave 3** *(blocked on Wave 2 completion)*
- [x] 02.1-03-PLAN.md — UI on operations: geometry tree, Operations panel, hammer, tags/filter, bulk+confirm

**Out of scope**: kerf redraw bug (separate PR: repaint only when not dragging); Phase 3 output; per-frame `Document::clone` latency.

### Phase 3: Per-Operation Laser Output
**Goal**: A persisted job emits laser G-code per operation — power, feed, and passes — with the same shape cut in multiple ordered passes and identical bytes from every surface.
**Mode:** mvp
**Depends on**: Phase 2.1
**Requirements**: OUT-01, OUT-02, OUT-03, OUT-04, OUT-05
**Success Criteria** (what must be TRUE):
  1. A job containing Cut, Engrave, and Score layers emits each operation's own power, feed, and pass count in the G-code.
  2. A shape assigned to both an Engrave and a Cut layer emits two ordered passes over the same geometry.
  3. The same persisted job exported from desktop, CLI, and MCP produces byte-identical G-code.
  4. Reopening a saved job and re-exporting reproduces the same operation order and identical bytes.
  5. A job containing unassigned shapes emits none of their geometry.
**Plans**: 3 plans
**Wave 1**
- [ ] 03-01-PLAN.md — resolve + operation-major emit_job + CLI rewiring + golden tests (tracer)

**Wave 2** *(blocked on Wave 1 completion)*
- [ ] 03-02-PLAN.md — CLI-vs-core byte parity + save/load round-trip tests

**Wave 3** *(blocked on Wave 2 completion)*
- [ ] 03-03-PLAN.md — resolve integration tests, pass clamping, hostile-input coverage

### Phase 4: Job Planning and Laser Features
**Goal**: Job ordering measurably reduces travel while laser-specific features — tabs, overcut, and focus/Z — are represented and emitted.
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: PLAN-01, PLAN-02, PLAN-03
**Success Criteria** (what must be TRUE):
  1. The orderer applies 2-opt with start-point selection, and a regression test on a fixed job reports lower travel distance than the greedy baseline.
  2. A laser job configured with tabs and overcut emits them at the configured positions in the G-code.
  3. Focus/Z height is configurable per operation and emitted per pass in the job output.
**Plans**: TBD

## Next Milestone (Not in This Roadmap)

The following requirements are deferred to a separate GSD milestone after this
document-organization product ships:

- **STREAM-01..04**: byte-accurate GRBL in-flight window, `GcodeProtocol` integration, pause/resume/abort, mock transport.
- **RECOVERY-01..02**: durable checkpoint, beam-safe resume after reconnect.
- **RASTER-01..03**: threshold/dither/DPI/overscan/joinPixel/bidirectional parity on golden images.

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 2.1 → 3 → 4
(Phase 3 depends on the model from Phase 2.1; execution stays sequential.)

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Operation-First Document Model | 4/4 | Complete | 2026-11-24 |
| 2. Layer/Operations UI and Assignment | 5/5 | Complete | 2026-11-24 |
| 2.1. Geometry/Operations Separation (INSERTED) | 3/3 | Complete    | 2026-11-25 |
| 3. Per-Operation Laser Output | 0/3 | Planned | - |
| 4. Job Planning and Laser Features | 0/TBD | Not started | - |
