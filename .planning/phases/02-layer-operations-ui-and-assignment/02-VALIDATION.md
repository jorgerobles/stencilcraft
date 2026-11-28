---
phase: "2"
slug: "layer-operations-ui-and-assignment"
status: draft
nyquist_compliant: true
wave_0_complete: false
created: "2026-11-24"
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Rust built-in test harness (`cargo test`) |
| **Config file** | none — `xtask/src/main.rs` defines the gates |
| **Quick run command** | `cargo test -p stencilcraft-engine --lib --locked` (from worktree root) |
| **Full suite command** | `cargo xtask ci` (fmt + clippy -D warnings + workspace tests) |
| **Estimated runtime** | ~60–180 seconds (quick), several minutes (full) |

---

## Sampling Rate

- **After every task commit:** Run `cargo test -p stencilcraft-engine --lib --locked`
- **After every plan wave:** Run `cargo test -p stencilcraft-engine -p stencilcraft-ui-egui --locked`
- **Before `/gsd-verify-work`:** `cargo xtask ci` must be green; also `cargo test -p craft-stencilcraft-format -p craft-stencilcraft-document --locked` in `refs/craft-libs` if the model is touched
- **Max feedback latency:** 180 seconds

---

## Per-Task Verification Map

Task IDs are assigned by the planner; requirement-level rows below are the contract to map onto them.

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 02-02-T1/T2, 02-04-T1 | 02-02/02-04 | 2/4 | UI-01 | — | Add layer creates `operation: None` (no implicit Cut) | unit | `(cd …/worktrees/stencilcraft-planning && cargo test -p stencilcraft-engine layer_add_is_operationless --locked)` | ❌ W0 | ⬜ pending |
| 02-02-T2, 02-04-T1 | 02-02/02-04 | 2/4 | UI-02 | — | `LayerSetOperation` sets/clears; `DocumentNew` has no Cut | unit | `(cd …/stencilcraft-planning && cargo test -p stencilcraft-engine document_new_has_no_cut --locked)` | ❌ W0 | ⬜ pending |
| 02-02-T2, 02-04-T2/T3 | 02-02/02-04 | 2/4 | UI-03 | T-02-01 | Add/remove membership is additive, geometry unchanged, locked layer rejected | unit | `(cd …/stencilcraft-planning && cargo test -p stencilcraft-engine assign_preserves_geometry --locked)` | ❌ W0 | ⬜ pending |
| 02-02-T1/T2, 02-05-T* | 02-02/02-05 | 2/5 | UI-04 | — | `LayerMove` changes `production_sequence()`; Pass N derived | unit | `(cd …/stencilcraft-planning && cargo test -p stencilcraft-engine reorder_changes_pass_order --locked)` | ❌ W0 | ⬜ pending |
| 02-03-T2, 02-05-T* | 02-03/02-05 | 3/5 | UI-05 | — | Colour/visible/lock round-trip; paint uses layer colour | unit | `cargo test -p stencilcraft-engine layer_properties_roundtrip --locked` + `cargo test -p stencilcraft-ui-egui layer_colour_affects_visual --locked` (both cd worktree) | ❌ W0 | ⬜ pending |
| 02-03-T1/T2 | 02-03 | 3 | UI-06 | — | `unassigned_shapes()` excludes operation-layer members; UI rows derived | unit | `(cd …/refs/craft-libs && cargo test -p craft-stencilcraft-document unassigned --locked)` + `(cd …/worktrees/… && cargo test -p stencilcraft-ui-egui unassigned_rows --locked)` | ✅ model / ❌ UI | ⬜ pending |
| 02-03-T1/T3 | 02-03 | 3 | UI-07 | — | Effective operation + source computed from layer/memberships | unit | `(cd …/worktrees/stencilcraft-planning && cargo test -p stencilcraft-ui-egui effective_operation --locked)` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] Worktree re-pin to Phase 1 v2 rev + v1→v2 call-site migration (compile gate before any UI work)
- [ ] `crates/engine` tests for membership/operation/order commands (geometry invariance, additive semantics, locked-layer guard)
- [ ] `crates/ui-egui` pure helpers extracted for testing: `build_layer_rows(doc)`, `pass_number(doc, index)`, `effective_operation(doc, selection)`, layer-colour-aware `shape_visual`
- [ ] No new test framework install needed — Rust harness and `xtask ci` already exist

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Visual confirmation that the Layers tree, pass badges, and colour swatches render as intended | UI-01..UI-07 | Headless environment has no display | Run `cargo run -p stencilcraft` on a GUI host; add layers, assign objects, reorder, toggle eye/lock, change colour |

*All non-visual behaviors have automated verification.*

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 180s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
