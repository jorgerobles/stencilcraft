# Phase 3: Per-Operation Laser Output — Research

**Researched:** 2026-11-24
**Domain:** Per-operation G-code emission (power/feed/passes), multi-pass ordered output, cross-surface byte parity
**Confidence:** HIGH (all in-repo claims verified by reading source this session)

## Summary

LaserCraft already has a deterministic, pure G-code emitter (`lasercraft-core::emit`) and a persisted job model (`JobFile` with `operations`, `bindings`, `toolpaths`), but nothing connects them: `emit()` takes a flat `&[Toolpath]` plus a single global `EmitOptions { passes }`, so power/feed can only vary per toolpath, passes are job-global, and operations/bindings in the job file are never read during emission. Phase 3's work is a **resolve step** (job + design → ordered, per-operation toolpath groups) feeding the existing emitter with per-operation parameters, plus a parity guarantee that desktop, CLI, and MCP all call the exact same pure function with the same inputs.

**Primary recommendation:** Add a pure `resolve`/`layout` function in `lasercraft-core` that turns a `JobFile` (plus its design's shapes) into an ordered `Vec<OperationBlock { operation, toolpaths }>`, then extend `emit` to iterate operation-major (operation order → per-operation passes → intra-operation toolpath order), emitting each operation's own `S`/`F`/pass count. Keep all three surfaces on this one function and one serialization path; byte parity then falls out of determinism (no HashMap iteration, no timestamps, fixed float formatting) rather than needing cross-surface tests to fix drift.

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Operation→toolpath resolution (bindings, membership, unassigned filtering) | lasercraft-core (pure) | — | No I/O in core; both CLI and future desktop/MCP call it |
| G-code byte emission | lasercraft-core::emit | — | Already pure and deterministic; extend, don't duplicate |
| Job file read/write (JSON) | lasercraft-io | — | The only place bytes are touched |
| CLI surface (`lasercraft emit/validate`) | lasercraft-cli | — | Thin shell over io + core |
| Desktop surface (`lasercraft-app`, not yet built) | core emit via same resolve path | lasercraft-io for save | PRD §7 names it; must not reimplement emission |
| MCP surface (`lasercraft-mcp`, not yet built) | core emit via same resolve path | — | Sibling pattern exists in stencilcraft-planning `crates/mcp` (`export.gcode` tool calls the shared export fn) |
| Machine profile | lasercraft-core::machine | persisted via CLI `--machine` JSON | Single `MachineProfile` type shared by all surfaces |

## Current Architecture for G-Code Generation

All verified by reading source this session.

**`crates/lasercraft-core/src/ir.rs`** — the persisted job model:
- `JobFile { version, design: DesignRef, operations: Vec<Operation>, bindings: Vec<Binding>, toolpaths: Vec<Toolpath> }` (ir.rs:126-136). `Operation { id, kind: ToolpathKind, feed, power, passes, kerf, tabs, overcut, focus_z, air }` (ir.rs:77-94) already carries **per-operation feed, power, and passes** — the data exists; it is simply never consumed by the emitter.
- `Binding { selector: Selector, operation: String }` with `Selector::{Node, Swatch, Layer}` (ir.rs:100-113) is the assignment edge from design objects to operations.
- `Toolpath { points, kind, closed, feed, power, source }` (ir.rs:44-60) carries optional per-path `feed`/`power` overrides.
- `ToolpathKind::{Cut, Engrave, Score, Raster}` (ir.rs:32-40), serde `snake_case`.

**`crates/lasercraft-core/src/emit.rs`** — the deterministic emitter:
- `emit(paths: &[Toolpath], profile: &MachineProfile, options: &EmitOptions) -> Result<String, EmitError>` (emit.rs:46-74). Structure: `; LaserCraft` header → profile header block → optional `$H` → `G21\nG90\n` → **one outer loop `for pass in 1..=options.passes` over ALL toolpaths** (emit.rs:62-67) → footer. Passes are currently global, not per-operation.
- Per toolpath: `G0` rapid to first point, `M4`/`M3` + optional ` S{power}` from `toolpath.power` via `profile.power_to_s`, `G1` moves with `F` emitted once on the first `G1`, per-point `S` overrides, closing move appended when `closed`, `M5` at end (emit.rs:133-171).
- `num()` (emit.rs:201-216) formats floats with fixed decimals then trims trailing zeros and normalizes `-0` → `0` — this is the determinism anchor for byte parity.
- `MAX_PASSES = 10` (emit.rs:11); validation rejects non-finite points, power outside 0..=100, feed outside `0..=max_feed`, `decimals > 6`, `s_min >= s_max` (emit.rs:76-131).

**`crates/lasercraft-core/src/machine.rs`** — `MachineProfile { name, work_area_mm, s_min, s_max, max_feed_mm_min, rx_buffer_bytes, laser_mode, decimals, header, footer, home_on_start }` with safe `Default` (s_max 1000, decimals 3, M4/dynamic) (machine.rs:28-66). `power_to_s(percent)` maps 0–100 onto `[s_min, s_max]` with `.round()` (machine.rs:71-75).

**`crates/lasercraft-io/src/lib.rs`** — `load_job`/`parse_job`/`save_job`; `save_job` uses `serde_json::to_string_pretty` + trailing newline (lib.rs:47-55). The only I/O boundary.

**`crates/lasercraft-cli/src/main.rs`** — `lasercraft emit --job --out --passes --machine` and `validate`; loads job + profile (default if `--machine` absent, main.rs:104-113) and calls `emit(&job.toolpaths, &profile, &EmitOptions { passes })` (main.rs:76). Note: the CLI `--passes` flag currently overrides passes globally — Phase 3 must reconcile this with per-operation `Operation.passes`.

**`crates/lasercraft-grbl/src/lib.rs`** — streaming `BufferWindow` only; no G-code generation. Not in scope for Phase 3 output bytes.

**Gap (the phase's core work):** `JobFile.operations` and `JobFile.bindings` are deserialized but never used. Nothing resolves bindings → toolpaths, nothing orders operations, nothing applies `Operation.{feed,power,passes}`. `emit()` cannot express "Engrave pass over shape A, then Cut passes over shape A" because it has one global pass loop.

## Operations in the craft-libs Document Model (Phase 1 foundation)

From `refs/craft-libs/crates/stencilcraft-document/src/production.rs` (read this session):

- `Operation` is a data-carrying enum: `Cut(OperationSettings)`, `Engrave(OperationSettings)`, `Score(OperationSettings)`, plus KissCut/DieCut/Draw/Perforate/PrintThenCut/Punch/Guide (production.rs:89-110). `kind_name()` gives kebab-case wire names (`"cut"`, `"engrave"`, `"score"`, …) (production.rs:161-174).
- `OperationSettings { power: f64, speed_mm_per_min: f64, passes: u32, focus_mm: f64, kerf_mm: f64, tabs: Vec<Tab>, tip: Option<Tip>, deboss: bool, punch_diameter_mm: Option<f64> }` (production.rs:53-67) — the per-operation parameter snapshot (locked decision D-09). `Operation::validate()` rejects negative power/speed/kerf and `passes < 1` (production.rs:177-211).
- `Process`/`Strategy` two-axis model (production.rs:16-39): laser ops map `Cut/Engrave → Process::Laser`, `Score → Process::Creaser` (production.rs:129-139).
- `Layer { operation: Option<Operation>, members: Vec<u64>, ... }` (lib.rs:95-114): `produces()` = has a non-Guide operation; `is_output()` = visible && produces (lib.rs:122-136). **Layer order is the production sequence (D-19); intra-layer `members` order is preserved (D-20).**
- N:M membership: geometry stored once in `Document::shapes`, layers reference by id (D-02). "Unassigned" is derived — shapes in no operation layer (D-04). `Shape::cut_path()` strips hidden contours (production.rs:355-375).
- `refs/craft-libs/crates/stencilcraft-format/src/lib.rs`: versioned envelope — `FORMAT_NAME = "stencilcraft"`, `FORMAT_VERSION = 2`, `SUPPORTED_VERSIONS = &[1, 2]`, `HEADER_KEY = "stencilcraft"`, and `DocumentFields` owns all wire field names with serde defaults/`skip_serializing_if` so old files load and default fields stay out of output (lib.rs:31-93).

**Mapping to LaserCraft's IR:** LaserCraft's `ir::Operation` is the flatter laser-only twin of the document's `Operation`: `kind: ToolpathKind` ↔ `Operation` variant, `feed` ↔ `speed_mm_per_min`, `power`, `passes`, `focus_z` ↔ `focus_mm`, `kerf`, `tabs`, `overcut`. Phase 3 should resolve the *document* (layers with operations, N:M members) into the *job* (`operations` + `bindings` + `toolpaths` with per-path `source` ids), then emit from the job — the job file remains the single source of truth for output (PRD §6, docs/PRD.md:122-130).

## The Three Surfaces and Byte Parity (OUT-03)

- **CLI** exists today (`lasercraft emit`). **Desktop** (`lasercraft-app`) and **MCP** (`lasercraft-mcp`) are named in the PRD as later crates (docs/PRD.md:137) and do not exist in `rustfactor/` yet. The sibling project shows the intended MCP pattern: `worktrees/stencilcraft-planning/crates/mcp/src/lib.rs` exposes an `"export.gcode"` tool (line 260) that calls the *same* shared export function the UI uses — one implementation, many front doors.
- **Parity strategy (prescriptive):**
  1. Exactly one emission entry point: `lasercraft_core::emit` (extended for per-operation blocks). No surface may format G-code itself.
  2. All surfaces resolve jobs through the same pure `resolve` in core and pass the *same* `MachineProfile`. Default profile is `MachineProfile::default()` when none is supplied — identical on every surface (main.rs:104-113).
  3. Determinism rules already in place and to be preserved: no `HashMap` iteration anywhere in the emit path (use `Vec` in persisted order); no timestamps/random ids in output; float formatting only via `num()`; job files serialized only via `serde_json` (`to_string_pretty` for saves, lib.rs:49) — serde_json/ryu float output is deterministic for identical `f64` bit patterns.
  4. `--passes` CLI flag becomes a *global override* (or is removed in favor of `Operation.passes`); whichever is chosen must be applied identically by all surfaces — simplest parity answer: surfaces never inject passes, they come only from the persisted `Operation.passes`.
  5. Parity test: golden-bytes test that runs the same job JSON through the CLI binary and the in-process emit and asserts `assert_eq!` on bytes (pattern already used in `crates/lasercraft-core/tests/emit.rs:24-61`, `golden_two_paths_one_pass`).

## Recommended Implementation Approach

1. **Resolve step in `lasercraft-core`** (new module, e.g. `layout.rs`, pure): `fn resolve_job(job: &JobFile, shapes: ...) -> Result<Vec<OperationBlock>, ResolveError>` producing operation blocks in **persisted order** (document layer order D-19 → job operation order), each with its member toolpaths in intra-layer order (D-20). Toolpaths whose `source` matches no binding/operation are dropped → **OUT-05** (unassigned shapes emit nothing).
2. **Operation-major emission:** change the pass loop from "passes × all paths" to "for each operation block (in order): for pass in 1..=operation.passes: emit its toolpaths with operation.power/operation.feed". This satisfies **OUT-01** (each operation's own S/F/passes) and **OUT-02** (same geometry appears once per operation block, ordered Engrave-then-Cut). Per-operation `S`/`F` should be emitted as toolpath-level `power`/`feed` (the emitter already supports these, emit.rs:151-153,160-162) — i.e., resolve stamps each toolpath with its operation's parameters, and `emit` barely changes. Operation comments (`; operation <id> (<kind>)`) between blocks aid debugging without breaking determinism.
3. **Keep `emit` signature stable where possible** — add `emit_job(&JobFile, &MachineProfile)` as the new canonical entry that resolves then emits; keep `emit(&[Toolpath], …)` for raw use. CLI/MCP/desktop call `emit_job`.
4. **OUT-04 (reopen → identical bytes):** rely on serde round-trip fidelity — `Vec` order is preserved, defaults via `#[serde(default)]` are stable, `skip_serializing_if` keeps files canonical, and float formatting is fixed-precision. Add a save→load→emit→compare golden test in `lasercraft-io/tests/` and `lasercraft-core/tests/`.
5. **Validation:** reuse `EmitError` variants; add resolve-level errors (unknown binding selector, operation with zero members, passes > MAX_PASSES).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| G-code bytes | a second writer per surface | `lasercraft-core::emit` | one deterministic implementation = byte parity by construction |
| Float formatting | `format!("{}", x)` per surface | `num()` in emit.rs | trims zeros, fixes `-0`, fixed decimals |
| Job JSON serialization | hand-rolled JSON | `serde_json` via lasercraft-io | ryu-backed, deterministic |
| Operation settings model | a third operation enum | document model's `Operation`/`OperationSettings` + flat `ir::Operation` mapping | Phase 1 locked D-06/D-09 |
| Passes semantics | per-surface override flags | persisted `Operation.passes`, clamped by `MAX_PASSES` | OUT-04 reproducibility |

## Common Pitfalls

1. **Global pass loop survives the change.** The current `for pass in 1..=options.passes` over all paths (emit.rs:62) would multiply every operation's geometry by the *job-global* pass count. Per-operation passes require operation-major nesting — the single most likely regression.
2. **HashMap/HashSet anywhere in resolution.** Operation order must come from the persisted `Vec` order (D-19/D-20), never from a map. Nondeterministic iteration breaks OUT-03/OUT-04 silently.
3. **Float drift on round-trip.** A job saved with `to_string_pretty` and reloaded must produce identical `f64`s; never route coordinates through `f32` or locale-aware formatting. `num()`'s `-0` → `0` normalization (emit.rs:211-214) must stay.
4. **`--passes` CLI flag vs `Operation.passes`.** If the flag silently overrides per-operation passes, CLI output diverges from desktop/MCP (OUT-03). Decide once: flag = global multiplier/override applied by resolve, or remove it.
5. **Bindings referencing missing shapes/operations.** Untrusted job files: resolve must return calm `Err`, never panic — core denies `unwrap`/`panic` (lib.rs:5-11). Dangling binding → error or skip, planner must pick; skipping silently hides data loss, erroring breaks OUT-04 on old files.
6. **Unassigned shapes leaking via `toolpaths`.** `JobFile.toolpaths` is a flat list; if emit reads it directly, unassigned geometry leaks (OUT-05 fails). Emit only resolved, bound toolpaths.
7. **Score-as-laser confusion.** Document model maps `Score → Process::Creaser`, but LaserCraft's `ToolpathKind::Score` is emitted as a laser pass. For Phase 3 all three of Cut/Engrave/Score emit laser G-code with their own S/F — confirm this matches user expectation before locking.

## Validation Architecture

`workflow.nyquist_validation` is enabled (`.planning/config.json`). Existing pattern: one integration binary per crate (`crates/lasercraft-core/tests/emit.rs` with golden-bytes asserts, `lasercraft-io/tests/job_io.rs`, `lasercraft-grbl/tests/window.rs`).

| Req | Behavior | Test | Command |
|-----|----------|------|---------|
| OUT-01 | per-operation S/F/passes in output | golden test: 3 ops with distinct params → assert distinct `S`/`F`/`; pass n` counts | `cargo test -p lasercraft-core` |
| OUT-02 | same shape in Engrave+Cut → two ordered blocks | golden test asserting block order and repeated geometry | `cargo test -p lasercraft-core` |
| OUT-03 | CLI vs in-process bytes identical | run CLI binary on fixture job, `assert_eq!` with `emit_job` output | `cargo test -p lasercraft-cli` (or shell golden in core tests) |
| OUT-04 | save→load→emit byte-identical | round-trip test through `lasercraft_io::save_job`/`load_job` | `cargo test -p lasercraft-io` + core |
| OUT-05 | unassigned shape absent | job with unbound toolpath → assert geometry coords absent from output | `cargo test -p lasercraft-core` |

Wave 0 gaps: new `layout.rs`/resolve tests file; cross-surface parity fixture job JSON; no new framework needed (std `assert_eq!` on bytes, matching existing style).

## Security Domain

`security_enforcement` is enabled (ASVS level 1). Job files and machine profiles are **untrusted input** (the document crate's rule: malformed files must never crash, stencilcraft-document lib.rs:3-5). Applicable: **V5 Input Validation** — all job/profile parsing goes through serde + `validate()` in emit.rs + resolve errors; core forbids `unwrap`/`expect`/`panic` (lib.rs:5-11, `unsafe_code = "forbid"` workspace-wide). No auth/session/crypto categories apply to offline G-code emission. MCP surface, when built, must follow the sibling project's automation rule: refuse file access outside explicit roots (craft-libs AGENTS.md rule 18).

## Environment Availability

| Dependency | Required By | Available | Notes |
|------------|-------------|-----------|-------|
| Rust toolchain (edition 2021, rust-version 1.80) | all crates | ✓ | workspace builds (`target/` present) |
| cargo test/clippy/fmt | validation | ✓ | standard workspace checks |
| Desktop app crate | OUT-03 surface | ✗ not yet built | `lasercraft-app` per PRD §7; parity by construction via shared core |
| MCP server crate | OUT-03 surface | ✗ not yet built | `lasercraft-mcp` per PRD §7; sibling pattern in stencilcraft-planning `crates/mcp` |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Desktop and MCP surfaces will be new thin crates over `lasercraft-core`, not reimplementations (PRD §7 names `lasercraft-app`/`lasercraft-mcp`) [ASSUMED] | Surfaces | If a surface reimplements emission, OUT-03 fails by design |
| A2 | Operation-major pass nesting (all passes of op 1, then all of op 2) is the intended reading of "two ordered passes over the same geometry" [ASSUMED] | Implementation | Alternative pass-major order (pass 1 of all ops, then pass 2) changes bytes; confirm with user |
| A3 | CLI `--passes` will become an override or be removed in favor of persisted `Operation.passes` [ASSUMED] | Pitfalls | Wrong choice silently breaks OUT-03 |

## Open Questions — RESOLVED

1. **Pass-major vs operation-major ordering** for multi-pass multi-operation jobs (A2) — **RESOLVED: operation-major** (matches "Cut is normally last", Phase 1 CONTEXT specifics). All passes of operation 1 complete before operation 2 begins.
2. **Dangling bindings**: **RESOLVED: error** with a clear message for OUT-04 safety. A job referencing a non-existent shape/operation binding fails loudly rather than silently skipping.
3. **`emit` signature**: **RESOLVED: new `emit_job` wrapper** — keeps existing golden tests valid while adding the operation-aware path.

## Sources

### Primary (HIGH confidence — read this session)
- `rustfactor/crates/lasercraft-core/src/{ir.rs,emit.rs,machine.rs,lib.rs}` — job model, emitter, machine profile
- `rustfactor/crates/lasercraft-cli/src/main.rs` — CLI surface
- `rustfactor/crates/lasercraft-io/src/lib.rs` — job I/O
- `rustfactor/crates/lasercraft-core/tests/emit.rs` — golden-test pattern
- `refs/craft-libs/crates/stencilcraft-document/src/{lib.rs,production.rs}` — Layer/Operation/OperationSettings model
- `refs/craft-libs/crates/stencilcraft-format/src/lib.rs` — versioned envelope, FORMAT_VERSION=2
- `.planning/REQUIREMENTS.md`, `.planning/ROADMAP.md`, `.planning/config.json`, `.planning/phases/01-…/01-CONTEXT.md` — requirements, locked decisions, config
- `docs/PRD.md` — crate layout and JobFile contract (superseded v0.1 draft, architecture sections still authoritative for crate plan)
- `worktrees/stencilcraft-planning/crates/mcp/src/lib.rs:260` — `export.gcode` MCP tool pattern

## Metadata

**Confidence breakdown:**
- Current architecture: HIGH — all files read this session, quoted with line ranges
- Document-model patterns: HIGH — craft-libs sources read this session
- Surface parity strategy: MEDIUM — desktop/MCP crates don't exist yet; strategy is prescriptive by design
- Pass-ordering semantics: LOW — flagged as Assumption A2, needs user confirmation

**Research date:** 2026-11-24
**Valid until:** 2026-12-24
