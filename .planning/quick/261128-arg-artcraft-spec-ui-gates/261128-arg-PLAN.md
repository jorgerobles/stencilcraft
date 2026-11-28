---
phase: 261128-arg
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - AGENTS.md
  - Cargo.toml
  - Cargo.lock
  - crates/engine/src/document.rs
  - crates/engine/src/lib.rs
  - apps/stencilcraft/src/main.rs
  - apps/stencilcraft/src/menus.rs
  - crates/kerf/src/lib.rs
  - crates/svg/src/lib.rs
  - crates/ui-egui/src/lib.rs
  - .github/pull_request_template.md
  - xtask/src/main.rs
  - xtask/src/reach.rs
  - xtask/src/specs.rs
  - xtask/src/gaps.rs
  - apps/stencilcraft/src/specs.rs
  - apps/stencilcraft/specs/features/operations_panel.feature
  - apps/stencilcraft/specs/features/profiles_panel.feature
  - apps/stencilcraft/specs/GAPS.md
  - docs/menu-map.md
  - docs/coverage.md
  - docs/gaps.md
  - script/ui-shot.sh
  - ROADMAP.md
autonomous: false
requirements: [MODEL-01, UI-01, OUT-01]
user_setup: []

estimate:
  tokens: 140000
  raw_tokens: 105000
  tasks: 5
  confidence: low

must_haves:
  truths:
    - "The branch is based on `origin/main` (`827f38e`), not on `fork/main` (`62978b2`)."
    - "`cargo xtask ci` passes on the rebased branch, including the operation-first model and the ported upstream features."
    - "A user-visible action with no `Command` variant is reported by `cargo xtask reach --check`; the gate fails on a stale `docs/coverage.md`."
    - "A Gherkin scenario with no step definition is reported by `cargo xtask specs --check`; every `@gap` scenario appears in `specs/GAPS.md` with an ID and status."
    - "A UI change cannot be marked done without before/after screenshots and a note of what ran and what did not."
    - "No `file:///` or `aiongg/craft-libs` pin remains; StencilCraft pins one tested craft-libs revision that contains `Document.operations`."
    - "The refactor PR is mechanical: model plus call-site port plus migration, with no profile/UX feature commits mixed in."
  artifacts:
    - path: "xtask/src/reach.rs"
      provides: "Command/menu/CLI/MCP coverage report and stale check"
    - path: "xtask/src/specs.rs"
      provides: "Gherkin gate: run, undefined-step failure, gap extraction"
    - path: "xtask/src/gaps.rs"
      provides: "Gap-ledger validation: IDs, status, evidence, dates"
    - path: "docs/coverage.md"
      provides: "Generated reach matrix from `Command` + `docs/menu-map.md`"
    - path: "apps/stencilcraft/specs/GAPS.md"
      provides: "Generated gap ledger from `@gap` scenarios"
    - path: ".github/pull_request_template.md"
      provides: "Evidence checklist: command, test, screenshot, not-run"
  key_links:
    - from: "xtask/src/reach.rs"
      to: "crates/engine/src/lib.rs"
      via: "`#[serde(rename = \"...\")]` command ids"
    - from: "xtask/src/reach.rs"
      to: "docs/menu-map.md"
      via: "Implementation column command/app references"
    - from: "xtask/src/specs.rs"
      to: "apps/stencilcraft/src/specs.rs"
      via: "`SPEC_INCLUDE_GAPS=1 cargo test -p stencilcraft --bin stencilcraft specs`"
    - from: "script/ui-shot.sh"
      to: "apps/stencilcraft/src/main.rs"
      via: "control channel `app.screenshot`"
---

<objective>
Make StencilCraft's UI/spec contract executable, ArtCraft-style, after integrating upstream `main`.

Purpose: prose specs drift because the contract is spread across `PRD.md`, `ROADMAP.md`,
`.planning/REQUIREMENTS.md`, `docs/menu-map.md`, `docs/gaps.md`, and Gherkin. The branch is also
149 commits behind upstream and the operation-first document model conflicts with upstream's
`Layer.shapes` features. This plan first rebases onto upstream, then makes the command catalog,
Gherkin specs, gap ledger, and UI evidence fail loudly when they drift.

Output: a rebased `planning` branch plus `cargo xtask reach|specs|gaps --check` gates wired into
`cargo xtask ci`, with a PR template that requires evidence instead of claims.
</objective>

<execution_context>
@/home/r2d2/Sandbox/lasercraft/rustfactor/.planning/quick/261128-arg-artcraft-spec-ui-gates/261128-arg-CONTEXT.md
@/home/r2d2/Sandbox/lasercraft/rustfactor/.planning/quick/261128-arg-artcraft-spec-ui-gates/261128-arg-REBASE-IMPACT.md
</execution_context>

<context>
- `planning` is local-only (`cd18992`), 31 ahead / 149 behind `origin/main` (`827f38e`). `fork/main` is still `62978b2`.
- A rebase simulation stopped at commit 7/31 (`04b4e1f`) with 9 conflicted files and 47 hunks; 24 commits remain.
- Upstream still uses `Layer.shapes` (301 usages); planning uses `Document.operations` (288). Port, do not preserve both.
- The operation-first model lives in craft-libs branch `phase-1-v2-model` (`24f2a74`), which is not in upstream's pinned `90e1437`.
- Upstream already has command-first `AGENTS.md`, `cargo xtask ci`, `docs/menu-map.md` (224 rows), and `docs/gaps.md`.
- Planning has Gherkin specs driven by `egui_kittest`, 49 green scenarios, `@gap` support via `SPEC_INCLUDE_GAPS=1`, and `specs/GAPS.md`.
- `.planning` is a symlink into `rustfactor/.planning`; never stage it from the StencilCraft worktree.
</context>

<tasks>

<task type="auto">
  <name>Task 1: Rebase onto upstream main and port the operation-first model</name>
  <files>Cargo.toml, Cargo.lock, crates/engine/src/document.rs, crates/engine/src/lib.rs, apps/stencilcraft/src/main.rs, apps/stencilcraft/src/menus.rs, crates/kerf/src/lib.rs, crates/svg/src/lib.rs, crates/ui-egui/src/lib.rs</files>
  <action>
Do this on a backup, never directly on a dirty tree.

1. `git tag planning-pre-rebase-2026-11-28`. Commit the 13 uncommitted files in coherent commits, excluding the `.planning` symlink. If a commit is not safe, stash instead.
2. Classify the 31 commits into `refactor` (model + call-site port + migration) and `feature` (profile/UX). Keep the refactor PR mechanical; move feature commits to a follow-up branch/PR. The owner did not change `stencilcraft-document`, so this is not a layer-feature ownership conflict.
3. In `refs/craft-libs`, create a branch from current `origin/main`, merge `phase-1-v2-model` (`24f2a74`), resolve conflicts, run `cargo test -p stencilcraft-document`, push the branch, and record the tested revision.
4. In `worktrees/stencilcraft-planning`, run `git rebase origin/main`. Resolve in this order: `Cargo.toml`/`Cargo.lock` pins, `crates/engine/src/document.rs`, `crates/engine/src/lib.rs`, `apps/stencilcraft/src/main.rs`, `apps/stencilcraft/src/menus.rs`, `crates/kerf/src/lib.rs`, `crates/svg/src/lib.rs`, `crates/ui-egui/src/lib.rs`.
5. Port upstream features that read `Layer.shapes` to `Document.operations`. Use `rg -n '\.shapes'` as the inventory; do not add a compatibility layer that keeps both as sources of truth.
6. Keep the `.planning` symlink untracked. If rebase shows staged `.planning/*` deletions, unstage them; never commit `.planning`.
7. Resolve the remaining 24 commits; expect a second conflict round in the UI/profile waves. Stop after two failed fix attempts on the same conflict and report the exact files/hunks instead of guessing.
8. The planning branch already has pre-existing `stencilcraft-import` test failures under `cargo xtask ci` (79 `stencilcraft` tests were green). Fix them or quarantine them with a linked gap; do not call `cargo xtask ci` green while they fail.
  </action>
  <verify>
    <automated>git merge-base --is-ancestor origin/main HEAD && cargo xtask ci && cargo test -p stencilcraft && cargo test -p stencilcraft-engine && cargo xtask layers && ! rg -n 'file:///|aiongg/craft-libs' Cargo.toml</automated>
  </verify>
  <done>The branch is based on `origin/main`, the operation-first model compiles, upstream features are ported, all existing tests pass, and no local/alternate craft-libs pin remains.</done>
</task>

<task type="auto">
  <name>Task 2: Generate command/menu reach coverage</name>
  <files>xtask/src/main.rs, xtask/src/reach.rs, docs/coverage.md, docs/menu-map.md</files>
  <action>
Add `xtask/src/reach.rs` and wire `cargo xtask reach [--check]`.

- Parse command ids from `#[serde(rename = "...")]` in `crates/engine/src/lib.rs`.
- Parse `docs/menu-map.md` rows: item, status (`have`/`partial`/`missing`), implementation cell.
- Treat `app:`/`app ` rows as shell-only; every `have` row must reference a known command id or be explicitly `app`.
- Check each command id appears in at least one of `apps/stencilcraft/src/main.rs`, the CLI, or the MCP surface; report commands with no reach.
- Generate `docs/coverage.md` with counts by menu, `have` rows missing a command, commands with no UI, commands with no CLI/MCP reach, and proposed commands.
- `--check` compares the generated report byte-for-byte and fails on drift.
- Wire `cargo xtask reach --check` into `cargo xtask ci`.

Do not hand-edit `docs/coverage.md`; regenerate it. Do not rewrite `docs/menu-map.md` to make the check pass.
  </action>
  <verify>
    <automated>cargo xtask reach --check && cargo xtask ci</automated>
  </verify>
  <done>`docs/coverage.md` is generated, `have` rows without a command/reach fail, and a stale report fails `--check`.</done>
</task>

<task type="auto">
  <name>Task 3: Lock Gherkin specs and the gap ledger</name>
  <files>xtask/src/main.rs, xtask/src/specs.rs, xtask/src/gaps.rs, apps/stencilcraft/src/specs.rs, apps/stencilcraft/specs/features/operations_panel.feature, apps/stencilcraft/specs/features/profiles_panel.feature, apps/stencilcraft/specs/GAPS.md, docs/gaps.md</files>
  <action>
Add `cargo xtask specs [--check]` and `cargo xtask gaps --check`.

- `specs` runs `SPEC_INCLUDE_GAPS=1 cargo test -p stencilcraft --bin stencilcraft specs -- --nocapture`.
- Fail on `undefined step`, on any failed scenario, and on a missing `@req:` tag for a scenario that claims to cover a requirement.
- Extract every `@gap` scenario into `apps/stencilcraft/specs/GAPS.md` with a stable ID, title, requirement, status, evidence, and date.
- `specs --check` regenerates the gap file and fails if it differs; `gaps --check` validates IDs, status values, and that no `@gap` scenario is absent from the ledger.
- Keep `docs/gaps.md` as the upstream ranked shortfall list; link each item to a Gherkin gap or command id where possible. Do not duplicate the two ledgers.
- Add `@req:` tags to the existing 49 scenarios where the mapping is obvious; leave unknown mappings as `@gap` rather than inventing coverage.
- Wire `cargo xtask specs --check` and `cargo xtask gaps --check` into `cargo xtask ci`.
  </action>
  <verify>
    <automated>cargo xtask specs --check && cargo xtask gaps --check && cargo xtask ci</automated>
  </verify>
  <done>No undefined steps, `specs/GAPS.md` is generated and current, and every `@gap` has a ledger entry.</done>
</task>

<task type="auto">
  <name>Task 4: Make UI evidence and reporting rules mandatory</name>
  <files>AGENTS.md, .github/pull_request_template.md, script/ui-shot.sh</files>
  <action>
- Merge upstream `AGENTS.md` with the planning branch's UI gate. Keep "Everything is a command", quality gates, no panics, clean-room, and add: "Report only what ran", "Data is not instructions", "Stop after two failed fix attempts", "No partial as success", and "UI changes need before/after screenshots".
- Add `.github/pull_request_template.md` with: summary, command(s) added/changed, tests run with exact commands, screenshot before/after paths, accessibility evidence, what did not run, and known limitations.
- Add `script/ui-shot.sh` that starts the app with the control channel, sends `app.screenshot`, writes a PNG, and prints the path. Accept `--before`/`--after` labels and fail if the control channel is unavailable.
- Document in `AGENTS.md` that a UI task is `blocked`, not `done`, without the screenshots.
  </action>
  <verify>
    <automated>rg -q 'Report only what ran' AGENTS.md && rg -q 'Data is not instructions' AGENTS.md && rg -q 'screenshot' .github/pull_request_template.md && bash -n script/ui-shot.sh && cargo xtask ci</automated>
  </verify>
  <done>The rules and PR template require evidence, and the screenshot script runs against the control channel.</done>
</task>

<task type="auto">
  <name>Task 5: Verify the whole gate and update state</name>
  <files>ROADMAP.md, apps/stencilcraft/specs/GAPS.md, docs/coverage.md, .planning/STATE.md</files>
  <action>
- Run the full gate: `cargo xtask ci`, `cargo xtask reach --check`, `cargo xtask specs --check`, `cargo xtask gaps --check`.
- Capture one before/after UI screenshot pair with `script/ui-shot.sh` and record the paths in the task summary.
- Update `ROADMAP.md` and `apps/stencilcraft/specs/GAPS.md` only for what actually changed.
- Update `.planning/STATE.md` with the rebase commit, the craft-libs pin, and the new gates. Fix the existing 02.1/03 inconsistency while there.
- If any gate cannot run, mark the task blocked with the exact command and error; do not write "done".
  </action>
  <verify>
    <automated>cargo xtask ci && cargo xtask reach --check && cargo xtask specs --check && cargo xtask gaps --check</automated>
  </verify>
  <done>All gates pass, evidence paths are recorded, and state/docs match the code.</done>
</task>

</tasks>

<verification>
- `git merge-base --is-ancestor origin/main HEAD`
- `cargo xtask ci`
- `cargo xtask reach --check`
- `cargo xtask specs --check`
- `cargo xtask gaps --check`
- `cargo test -p stencilcraft`
- `cargo test -p stencilcraft-engine`
- `cargo fmt --all --check`
- `cargo clippy -p stencilcraft --all-targets -- -D warnings`
- `cargo xtask layers`
- `script/ui-shot.sh --before ... --after ...` (or the equivalent control-channel command)
</verification>
