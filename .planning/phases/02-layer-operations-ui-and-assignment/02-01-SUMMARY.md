---
phase: 02-layer-operations-ui-and-assignment
plan: 01
subsystem: document-model
tags: [rust, cargo, v2-model, migration, compile-gate, craft-libs]

# Dependency graph
requires:
  - phase: 01-operation-first-document-model
    provides: the uncommitted Phase 1 v2 operation-first document model in refs/craft-libs
provides:
  - refs/craft-libs branch phase-1-v2-model at captured sha 4a2fada7cdc3c13d8db974752fae65419f36e920
  - worktree Cargo.toml/Cargo.lock pinning craft-raster-codecs, craft-stencilcraft-format, craft-stencilcraft-document, craft-dxf to the local file:// rev
  - v1→v2 migration of the twelve library crates behind a green compile gate
  - derived production_sequence() / unassigned_shapes() primitives available to UI-04/UI-06 via the pinned v2 model
affects: [02-02 engine/app/UI migration, UI-04 pass ordering, UI-06 unassigned group]

# Actuals (#2632)
actuals:
  tokens: 18500
  tasks: 1
  commits: 1

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "CutStyle kept only as legacy input; every writer maps it to Option<Operation> via a local op_of helper"
    - "Layer membership reads go through doc.get_shape(layer.members[i]) — geometry lives once in Document::shapes"

key-files:
  created: []
  modified:
    - refs/craft-libs crates/stencilcraft-document + stencilcraft-format (committed on phase-1-v2-model)
    - worktree Cargo.toml, Cargo.lock (four storytold pins re-pointed)
    - worktree crates/{doc,kerf,svg,tiling,dxf,gcode,layout,import,pdf,generate,testkit,studio}

key-decisions:
  - "Pinned all four storytold craft crates to captured sha 4a2fada via file:// with TODO(push) comments; aiongg pins untouched (D-14)"
  - "CutOp.operation is Option<Operation>; shape operation resolves from the owning layer (D-15 field map)"
  - "Worktree changes left uncommitted per plan; only the craft-libs branch commit and this SUMMARY are committed"

patterns-established:
  - "op_of(CutStyle) -> Option<Operation> helper duplicated per crate that accepts legacy style input"
  - "member(doc, layer, i) test helper pattern for v2 membership indexing"

requirements-completed: [UI-04, UI-06]

# Coverage metadata (#1602)
coverage:
  - id: D1
    description: "v2 document model published on branch phase-1-v2-model and pinned into the worktree via file:// rev with regenerated Cargo.lock"
    requirement: UI-04
    verification:
      - kind: automated
        ref: "git -C refs/craft-libs rev-parse --verify phase-1-v2-model -> 4a2fada7cdc3c13d8db974752fae65419f36e920"
        status: pass
      - kind: automated
        ref: "grep -v '^#' Cargo.toml | grep -c file:///home/r2d2/Sandbox/lasercraft/refs/craft-libs -> 4"
        status: pass
      - kind: automated
        ref: "cargo test -p craft-stencilcraft-format -p craft-stencilcraft-document --locked -> 67 passed, 0 failed"
        status: pass
    human_judgment: false
  - id: D2
    description: "Twelve library crates migrated to Layer.operation/Layer.members/Document.shapes and compiling green against the pinned v2 rev"
    requirement: UI-06
    verification:
      - kind: automated
        ref: "cargo check -p stencilcraft-doc -p stencilcraft-kerf -p stencilcraft-svg -p stencilcraft-tiling -p stencilcraft-dxf -p stencilcraft-gcode -p stencilcraft-layout -p stencilcraft-import -p stencilcraft-pdf -p stencilcraft-generate -p stencilcraft-testkit -p stencilcraft-studio --locked -> exit 0"
        status: pass
      - kind: automated
        ref: "rg 'shape.operation()|.production.operation|layer.shapes|.template' over the 12 library crates -> no non-test v1 access"
        status: pass
    human_judgment: false

# Metrics
duration: 45min
completed: 2026-11-24
status: complete
---

# Phase 02 Plan 01: Publish the v2 model, re-pin, and migrate the library crates Summary

**Operation-first v2 document model committed on `phase-1-v2-model` (4a2fada), worktree re-pinned to it via `file://`, and all twelve library crates migrated to `Layer.operation`/`Layer.members` behind a green `cargo check --locked` gate**

## Performance

- **Duration:** 45 min
- **Started:** 2026-11-24T18:48:00Z
- **Completed:** 2026-11-24T19:33:17Z
- **Tasks:** 1
- **Files modified:** 19 (craft-libs, committed) + 21 worktree files (uncommitted per plan)

## Accomplishments
- Committed the Phase 1 v2 model on `refs/craft-libs` branch `phase-1-v2-model` at `4a2fada7cdc3c13d8db974752fae65419f36e920` (19 files, `.codegraph/` excluded); craft-libs tests green (67 passed, 0 failed)
- Re-pointed the four storytold pins (craft-raster-codecs, craft-stencilcraft-format, craft-stencilcraft-document, craft-dxf) to the captured sha via `file://` with `TODO(push)` comments; `cargo update` recorded the local rev in Cargo.lock; aiongg pins untouched
- Migrated all 12 library crates: `Layer.kind`→`operation: Option<Operation>`, `Layer.shapes`→`members` + `Document.shapes`/`get_shape`, `Shape.style`/`ShapeProduction.operation` removed, `push_shape` two-arg, `CutOp.operation` now `Option<Operation>` resolved from the owning layer
- Compile gate green: `cargo check --locked` over the twelve library crates exits 0; no v1 field access remains outside legacy `CutStyle` input mapping

## Task Commits

1. **Task 1: Publish the v2 model, re-pin the worktree, migrate the library crates** - `4a2fada` (feat(stencilcraft): operation-first v2 document model, refs/craft-libs)

**Plan metadata:** this SUMMARY commit (rustfactor planning repo)

## Files Created/Modified
- `refs/craft-libs` (branch `phase-1-v2-model`): v2 model commit — committed
- worktree `Cargo.toml` / `Cargo.lock` — four pins re-pointed to `file:///home/r2d2/Sandbox/lasercraft/refs/craft-libs` rev `4a2fada...`
- worktree `crates/doc/src/templates.rs`, `crates/kerf/src/lib.rs`, `crates/svg/src/lib.rs` + `tests/golden.rs`, `crates/tiling/src/lib.rs` + `marks.rs`, `crates/dxf/src/lib.rs`, `crates/gcode/src/lib.rs`, `crates/layout/src/{lib,job,mats,print}.rs`, `crates/import/src/{lib,handlers,render}.rs` + `tests/corpus.rs`, `crates/pdf/src/lib.rs`, `crates/testkit/src/lib.rs`, `crates/studio/src/lib.rs` + `tests/corpus.rs` + `examples/studio_dump.rs` — v1→v2 migration (uncommitted per plan)

## Decisions Made
- Pinned by captured commit sha only (no branch/tag refs), per D-14 and threat T-02-05/T-02-SC
- `ShapeProduction.operation` reads replaced by owning-layer lookup; `CutOp.operation` widened to `Option<Operation>`
- Worktree left uncommitted intentionally: the plan forbids worktree commits; 02-02 owns engine/app/UI migration on top of this pin

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- 02-02 can migrate `crates/engine`, `crates/ui-egui`, `crates/mcp`, and `apps/*` against the stable local pin; `production_sequence()` and `unassigned_shapes()` are available for UI-04/UI-06
- Before any push: flip the four `TODO(push)` pins back to `https://github.com/storytold/craft-libs.git` at `4a2fada7cdc3c13d8db974752fae65419f36e920`

---
*Phase: 02-layer-operations-ui-and-assignment*
*Completed: 2026-11-24*

## Self-Check: PASSED
- craft-libs branch `phase-1-v2-model` exists at 4a2fada7cdc3c13d8db974752fae65419f36e920; working tree clean except untracked `.codegraph/`
- worktree Cargo.toml has exactly 4 non-comment `file://` lines; Cargo.lock records the captured sha and no 75c70fa6 entry for the four re-pointed crates
- `cargo check --locked` over the twelve library crates exits 0
- craft-libs `cargo test -p craft-stencilcraft-format -p craft-stencilcraft-document --locked`: 67 passed, 0 failed
