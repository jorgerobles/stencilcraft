# 261128-arg context

## Goal

Adopt ArtCraft-style executable contracts in StencilCraft after integrating upstream `main`:

- one authority for rules;
- one command catalog per user-visible action;
- generated coverage from that catalog;
- Gherkin UI contracts with an explicit gap ledger;
- screenshot/accessibility evidence for UI changes;
- no claim of success without a command that proves it.

## Why this exists

LaserCraft's specs drift because the active contract is spread across `PRD.md` (superseded), `ROADMAP.md`, `.planning/REQUIREMENTS.md`, `docs/gaps.md`, `docs/menu-map.md`, and Gherkin features. Traceability is hand-maintained, so a feature can be "done" in prose while no test or command reaches it.

ArtCraft's lock is executable, not prose: a command catalog plus generated parity plus tests plus evidence. StencilCraft already has the raw pieces — `Command` in `crates/engine/src/lib.rs`, `cargo xtask`, Gherkin via `apps/stencilcraft/src/specs.rs`, and 49 green scenarios — so extend those; do not invent a second spec system.

## Locked decisions

- **Rebase first.** The gates run on a branch based on `origin/main` (`827f38e`), not on the stale `planning` base.
- **One authority:** root `AGENTS.md`. `.planning` is a symlink into `rustfactor/.planning`; never commit it from the StencilCraft worktree.
- **Command-first stays upstream's rule:** a user-visible action needs a `Command` variant in `crates/engine/src/lib.rs` and a test; UI/MCP call `engine.run(command)`.
- **Gherkin is the UI contract:** `apps/stencilcraft/specs/features/*.feature` drives `egui_kittest`; `@gap` scenarios are allowed only when `SPEC_INCLUDE_GAPS=1` and must appear in `specs/GAPS.md`.
- **Generated coverage wins:** `docs/menu-map.md` + `Command` enum + CLI/MCP reach produce `docs/coverage.md`; a stale generated file fails `--check`.
- **Evidence is mandatory:** UI changes need before/after screenshots from the control channel (`app.screenshot`) plus a note of what ran and what did not.
- **One craft-libs pin:** merge/rebase `phase-1-v2-model` onto current craft-libs main, test it, pin the tested revision; remove `file:///` and `aiongg` pins.
- **Refactor PR is separate from feature PRs:** split into (1) craft-libs model PR, (2) StencilCraft mechanical port PR, (3) profile/UX feature PRs. The refactor PR must not carry profile UX changes.
- **No ownership conflict:** upstream never changed `crates/stencilcraft-document`; `Layer` is unchanged from the base through `origin/main`. The integration work is porting upstream call sites from `Layer.shapes` to `Document.operations`, not reconciling a competing layer-model change.
- **No silent partial success:** a task with a skipped gate is `blocked`, not `done`.

## Assumptions

- `planning` is local-only; rebasing it will not rewrite shared history.
- Upstream `origin/main` is `827f38e`; `fork/main` remains `62978b2`.
- The measured conflicts (9 files, 47 hunks at commit 7/31) are a lower bound; the remaining 24 commits can add more.
- The operation-first model lives in `stencilcraft-document`; upstream's 149 commits still use `Layer.shapes`.
- `cargo xtask ci` is the current quality gate; extend it rather than adding a parallel script.

## Out of scope

- New product features beyond what upstream already added.
- Hardware streaming, recovery, raster, or nesting algorithms.
- Rewriting upstream `docs/menu-map.md` by hand.
- Changing `fork` history or force-pushing anything.

## Canonical references

- `/home/r2d2/Sandbox/lasercraft/worktrees/stencilcraft-planning/AGENTS.md`
- `/home/r2d2/Sandbox/lasercraft/worktrees/stencilcraft-planning/crates/engine/src/lib.rs`
- `/home/r2d2/Sandbox/lasercraft/worktrees/stencilcraft-planning/apps/stencilcraft/src/specs.rs`
- `/home/r2d2/Sandbox/lasercraft/worktrees/stencilcraft-planning/apps/stencilcraft/specs/`
- `/home/r2d2/Sandbox/lasercraft/worktrees/stencilcraft-planning/docs/menu-map.md`
- `/home/r2d2/Sandbox/lasercraft/worktrees/stencilcraft-planning/docs/gaps.md`
- `/home/r2d2/Sandbox/lasercraft/worktrees/stencilcraft-planning/xtask/src/main.rs`
- `261128-arg-REBASE-IMPACT.md` (same directory)
