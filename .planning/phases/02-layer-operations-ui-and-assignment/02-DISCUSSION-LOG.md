# Phase 2: Layer/Operations UI and Assignment - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-11-24
**Phase:** 2-Layer/Operations UI and Assignment
**Areas discussed:** Add-layer operation picker, Assignment interaction, Layers panel structure, Reordering & production order

---

## Add-Layer Operation Picker

| Option | Description | Selected |
|--------|-------------|----------|
| Preset catalogue | Full cooked preset list (Laser cut/engrave/score per D-07) | |
| Short enum | Cut/Engrave/Score only | |
| Empty layer, pick later | Create operation-less layer first | |

**User's choice:** Free text — operation popup on a configurable shortcut (default shift+LMB) on the **Add Layer button** (not on layer rows; shift+click there conflicts with multiple layer selection), plus a hammer icon on the right of the layer opening the same popup; panel with draggable operations (Illustrator appearance); literal UI-01 "requires operation on add" rejected.

**Notes:** Plain "Add Layer" creates an operation-less layer (confirmed, option 1). Drag source is an operations palette in the panel (drag onto row assigns). Applying an operation to a layer with existing shapes includes all of them (Phase 1 D-03 confirmed).

---

## Assignment Interaction

| Option | Description | Selected |
|--------|-------------|----------|
| Context menu "Send to Layer…" | Submenu listing layers; one or many selected objects | ✓ |
| Drag objects onto layer row | Visual but less discoverable | |
| Both | Menu + drag | |

**User's choice:** Context menu "Send to Layer…" (replaces `todo!()` at menus.rs:236).

**Notes:** Removal via the same submenu with a remove entry. Assignment is additive (N:M, keeps existing memberships). Membership shown as badges on object rows in the panel.

---

## Layers Panel Structure

| Option | Description | Selected |
|--------|-------------|----------|
| Nested tree + Unassigned group | Layer rows expand to member objects; collapsible derived Unassigned group at bottom | ✓ |
| Two flat sections | Keep current Layers/Objects split | |

**User's choice:** Nested tree + Unassigned group.

**Notes:** Layer properties (colour, lock, visibility, settings, operation) edited inline on the row: eye, lock, colour swatch, operation badge + hammer icon. Unassigned group is derived, never persisted (D-04/UI-06).

---

## Reordering & Production Order

| Option | Description | Selected |
|--------|-------------|----------|
| Drag handles | Drag rows to reorder | ✓ |
| Up/down buttons | Explicit arrow buttons | |

**User's choice:** Drag handles.

**Notes:** Production order shown as "Pass N" badge on operation layers (position among `Some(operation)` layers, D-19); operation-less layers show no badge. Drag interactions unified: palette→row assigns operation, row-drag reorders.

---

## The agent's Discretion

- Exact egui widget composition and drag payload mechanics.
- Operations palette placement (panel section vs flyout).
- Colour picker widget.
- Inspector (UI-07) effective-operation display details.

## Deferred Ideas

None.
