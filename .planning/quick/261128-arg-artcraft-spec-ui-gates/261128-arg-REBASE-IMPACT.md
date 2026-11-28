# Upstream rebase impact — 2026-11-28

Checked in `worktrees/stencilcraft-planning` with `git fetch --all --prune`.
No working-tree changes were made; the conflict test ran in a detached temporary worktree and was removed.

## Remote state

| Ref | Commit | State |
|---|---|---|
| `origin/main` | `827f38e` | advanced `62978b2..827f38e` (149 commits) |
| `fork/main` | `62978b2` | unchanged |
| `planning` | `cd18992` | local only; 31 ahead / 149 behind `origin/main` |

`planning` is not pushed to `fork`, so a rebase is safe from a shared-history point of view.
The worktree has 13 uncommitted files plus the `.planning` symlink. Those changes are not in the rebase simulation and will add conflicts.

## What upstream added

- 70 files, `+27,188 / -3,281`.
- 34 app modules: Send, Library, Print, Nesting, `.studio3` import, text, preferences, trace, PixScan, document tabs, and more.
- High-risk overlaps: `apps/stencilcraft/src/main.rs` (`+8,405`), `crates/engine/src/lib.rs` (`+1,745`), `crates/ui-egui/src/lib.rs` (`+1,297`), `Cargo.toml`, `Cargo.lock`, `crates/engine/src/document.rs`, `crates/svg/src/lib.rs`, `crates/kerf/src/lib.rs`, `apps/stencilcraft/src/menus.rs`.

## Model impact

Upstream still uses the old document model:

- `Layer.shapes`: 301 usages on `origin/main`.
- `Document.operations`: 30 usages on `origin/main`.

Planning uses the new model:

- `Layer.shapes`: 94 usages on `planning`.
- `Document.operations`: 288 usages on `planning`.

So the rebase is not a sync. Every upstream feature added on `Layer.shapes` must be audited and ported to `Document.operations`. The compiler and the existing operation-first tests are the inventory; do not port by guesswork.

## Rebase simulation

Detached worktree at `planning`, then `git rebase origin/main`.

- Stopped at commit **7/31** (`04b4e1f`, operation-first v2 model).
- 24 commits remain.
- 9 conflicted files, 47 conflict hunks:

| File | Hunks |
|---|---:|
| `Cargo.lock` | 26 |
| `Cargo.toml` | 1 |
| `apps/stencilcraft/src/main.rs` | 11 |
| `apps/stencilcraft/src/menus.rs` | 3 |
| `crates/engine/src/document.rs` | 1 |
| `crates/engine/src/lib.rs` | 2 |
| `crates/kerf/src/lib.rs` | 1 |
| `crates/svg/src/lib.rs` | 1 |
| `crates/ui-egui/src/lib.rs` | 1 |

The remaining 24 commits include the UI/profile waves that touch `main.rs` and `menus.rs` again, so expect a second round of conflicts after the first resolution.

## craft-libs impact

Planning depends on an unpublished branch:

- `phase-1-v2-model` at `24f2a74` (2 commits on top of `249a41b`).
- `Cargo.toml` also has `file:///home/r2d2/.../refs/craft-libs` and `aiongg/craft-libs` pins.

Upstream pins `storytold/craft-libs` at `90e1437`, which does **not** contain `Document.operations`.

Divergence: `origin/main` is 10,776 commits ahead of the merge base; the phase branch is 2 ahead. There are no upstream changes to `crates/stencilcraft-document` since the merge base, so the operation-first change itself is tractable, but the branch must be merged onto current craft-libs main and its tests must pass before StencilCraft pins it.

Action: merge or rebase `phase-1-v2-model` onto current craft-libs main, run `cargo test -p stencilcraft-document`, push the branch, pin the tested revision in StencilCraft, and remove the `file:///` and `aiongg` pins.

## What this means for current development

1. Rebase first. Adding new gates on the stale base just creates more conflict surface.
2. Commit or stash the 13 uncommitted files before rebasing; tag a backup first.
3. Resolve in this order: dependency pins → `stencilcraft-document` → engine commands → UI/main/menus → tests/specs.
4. Port upstream features to `Document.operations`; do not restore `Layer.shapes` as a second source of truth.
5. Keep upstream's command-first `AGENTS.md`; add the planning branch's UI screenshot gate on top.
6. Upstream already has `docs/menu-map.md` (224 rows: 93 have, 94 missing, 12 partial) and `docs/gaps.md`; planning has 49 Gherkin scenarios and `specs/GAPS.md`. Those are the raw material for generated coverage; do not replace them with a new prose spec.

## Recommended sequence

```sh
# 0. safety
git tag planning-pre-rebase-2026-11-28
# commit or stash the 13 uncommitted files first

# 1. craft-libs
# merge phase-1-v2-model onto current craft-libs main, test, push, record rev

# 2. StencilCraft
git switch planning
git rebase origin/main
# resolve 9 files, then continue; expect conflicts in the remaining 24 commits

# 3. verify
cargo xtask ci
cargo test -p stencilcraft
cargo test -p stencilcraft-engine
```

If the rebase takes more than a day, split it into a dedicated phase (`2.2 Upstream integration`) instead of hiding it inside a quick task.
