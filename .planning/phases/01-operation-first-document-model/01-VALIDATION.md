---
phase: "1"
slug: "operation-first-document-model"
status: draft
nyquist_compliant: false
wave_0_complete: false
created: "2026-11-24"
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Rust built-in `#[test]` / `cargo test` (no external test crate) |
| **Config file** | none — inline `#[cfg(test)]` modules plus `tests/fixtures/` |
| **Quick run command** | `cargo test -p craft-stencilcraft-document -p craft-stencilcraft-format --locked` |
| **Full suite command** | `cargo test -p craft-stencilcraft-document -p craft-stencilcraft-format --all-targets --locked` then `cargo clippy -p craft-stencilcraft-document -p craft-stencilcraft-format --all-targets -- -D warnings` and `cargo fmt --check` |
| **Estimated runtime** | ~30 seconds (warm target cache) |

---

## Sampling Rate

- **After every task commit:** Run `cargo test -p craft-stencilcraft-document -p craft-stencilcraft-format --locked`
- **After every plan wave:** Run the full suite command above (tests + clippy `-D warnings` + `fmt --check`)
- **Before `/gsd-verify-work`:** Full suite must be green; migration fixtures pass
- **Max feedback latency:** 60 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| TBD | TBD | TBD | MODEL-01 | T-1-01 | Reject non-finite/negative geometry on save | unit/round-trip | `cargo test -p craft-stencilcraft-document round_trip --locked` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | MODEL-02 | — | N/A | unit | `cargo test -p craft-stencilcraft-document layer_round_trip --locked` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | MODEL-03 | — | N/A | unit | `cargo test -p craft-stencilcraft-document shape_in_two_layers --locked` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | MODEL-04 | T-1-02 | Panic-free migration on hostile v1 input | integration | `cargo test -p craft-stencilcraft-document migrate_v1 --locked` | ❌ W0 (fixtures exist) | ⬜ pending |
| TBD | TBD | TBD | MODEL-05 | — | N/A | unit | `cargo test -p craft-stencilcraft-document unassigned --locked` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | MODEL-06 | — | N/A | unit | `cargo test -p craft-stencilcraft-document operation_round_trip --locked` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | MODEL-07 | T-1-03 | Prune dangling membership ids on delete | unit | `cargo test -p craft-stencilcraft-document remove_layer --locked` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | (format) | T-1-04 | v3 rejected, unknown fields skipped safely | unit | `cargo test -p craft-stencilcraft-format --locked` | ✅ existing tests adapt | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `crates/stencilcraft-document/tests/it/main.rs` — integration binary covering MODEL-01..07
- [ ] Migration tests over `tests/fixtures/{starter,poster,stencil-frame}.stencilcraft` (MODEL-04)
- [ ] v2 round-trip test with a shape in two ordered layers (MODEL-01/03)
- [ ] Unassigned-derivation test (MODEL-05)
- [ ] `remove_layer` test (MODEL-07)
- [ ] Operation round-trip test incl. all 9 variants (MODEL-06)
- [ ] Format-crate version-negotiation tests (v1 accepted, v2 written, v3 rejected)
- [ ] No test framework install needed (built-in cargo)

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| N/A | — | All phase behaviors have automated verification | — |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 60s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
