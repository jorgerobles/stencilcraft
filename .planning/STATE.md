---
gsd_state_version: "1.0"
current_phase: 03
current_phase_name: Per-Operation Laser Output
status: executing
stopped_at: Phase 02.1 complete, ready to execute Phase 03
last_updated: "2026-11-25T14:01:06.215Z"
last_activity: 2026-11-25
last_activity_desc: Phase 02.1 complete, transitioned to Phase 03
state_head: 3e42d842202a49d30741b0d9223003ae2265c5fc
progress:
  total_phases: 5
  completed_phases: 3
  total_plans: 15
  completed_plans: 12
  percent: 60
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-11-24)

**Core value:** A persisted laser document can express ordered operations per object/layer and produce per-operation output identically from desktop, CLI, and MCP.
**Current focus:** Phase 02.1 — Geometry/Operations Separation

Note: the repo-root `ROADMAP.md` is StencilCraft's own product roadmap. This
file's roadmap is `.planning/ROADMAP.md` (the current GSD milestone plan:
document organization). Streaming, recovery, and raster are the next milestone.

## Current Position

Phase: 03 — Per-Operation Laser Output
Plans: 5 of 5 complete (Phase 2); 3 planned (Phase 2.1)
Status: Ready to execute
Last activity: 2026-11-25 — Phase 02.1 complete, transitioned to Phase 03

Next: Phase 2.1 Wave 1 (02.1-01 import ADR + geometry-preserving importer)

Progress: [██████░░░░] 60%

## Performance Metrics

**Velocity:**
- Total plans completed: 12
- Average duration: —
- Total execution time: —

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1. Operation-First Document Model | 4 | 4 | — |
| 2. Layer/Operations UI and Assignment | 5 | 5 | — |
| 2.1. Geometry/Operations Separation | 0 | 3 | — |
| 02.1 | 3 | - | - |

**Recent Trend:**
- Last 5 plans: —
- Trend: —

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Geometry stored once; operation layers reference shapes (N:M), replacing `Layer.shapes: Vec<Shape>`.
- Direct object operation is represented as a one-shape layer — no new per-shape operation ownership.
- "Unassigned" is a derived UI group, never persisted and never emitted.
- Document changes go to `craft-libs`; development pins a local branch via `[patch]`.
- Current milestone is document organization (phases 1–4); streaming/recovery/raster are the next milestone.
- Phase 2.1 (inserted): import preserves `<g>` nesting as groups; styling is a hint (ADR 0001), legacy mode behind `import_legacy_style_operations`.
- Phase 2.1 (inserted): operations are separated from geometry — `Document.operations` first-class, `Layer` geometry-only; unpublished v2 revised in place, v1 migrates in one hop.
- Bulk-create confirmation is preference-driven (`bulk_confirm_threshold`, default 10).
- Quick task `261127-pfl` (2026-11-27): operations/profiles link UX wave landed — profile `id`/colour, Photoshop link glyph, compact palette, member move/Shift-copy (`OperationMoveMembers`), profile columns/counter/footer icons and the differing-settings dialog; 49 Gherkin scenarios green. Docs in `OPERATIONS-LAYERS-REQUIREMENTS.md`, `locators.md`, `GAPS.md`, `ROADMAP.md`.
- Follow-up (2026-11-27): profile colours were parsed as decimal, so every `#rrggbb` fell back to grey. Fixed to `from_str_radix(hex, 16)` with a 6-char guard and per-kind fallback; profiles now share the operation colour popup (`ColorTarget`) and a committed colour recolours linked operations, row wells and the canvas. Visual gate captured; `cargo test -p stencilcraft` 79 green, clippy/fmt/layers clean (only pre-existing `stencilcraft-import` tests fail under `xtask ci`).

### Pending Todos

- [ ] `261128-arg` — Draft PR [#143](https://github.com/dylanmtaylor/stencilcraft/pull/143) opened from `fork/feature/operations-first` to `upstream/main`; branch is 32 commits, `CONFLICTING`, not rebased. PR #142 (`planning`) was closed as superseded. Next: rebase onto `origin/main` (`827f38e`), resolve the operation-first conflicts, then add executable reach/spec/gap/UI-evidence gates. Plan: `.planning/quick/261128-arg-artcraft-spec-ui-gates/261128-arg-PLAN.md`; impact: `261128-arg-REBASE-IMPACT.md`.

### Blockers/Concerns

- Cross-repo `[patch]` must be replaced by an upstreamable craft-libs revision before PR.
- `ShapeProduction.operation` users (layout grouping, `ShapesSetStyle`) need a migration path that does not silently drop per-shape overrides.
- Root `ROADMAP.md` and `docs/gaps.md` must move with any shipped StencilCraft behaviour.

## Deferred Items

Items acknowledged and deferred at milestone close, most recent first:

| Category | Item | Status | Deferred At | Milestone |
|----------|------|--------|-------------|-----------|
| *(none)* | | | | |

## Session Continuity

Last session: 2026-11-25T00:00:00.000Z
Stopped at: Phase 02.1 complete, ready to execute Phase 03
Resume file: .planning/phases/02.1-geometry-operations-separation/02.1-CONTEXT.md
