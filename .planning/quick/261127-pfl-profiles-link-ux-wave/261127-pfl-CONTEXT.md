# Quick Task 261127-pfl — Profiles link UX wave

## Task

Formalize the next Operations/Profiles UX wave: member rows dragged between operations,
a Photoshop-style profile link state, compact swatches, and the Profiles panel redesign
(columns, swatches, counts, footer icons).

## Locked decisions (user, 2026-11-27)

1. **Extend `Profile`, do not bump craft-libs.** Add a stable `id` and a `color` to
   `Profile`. Do not add `Operation.profile_id`. Links stay derived from
   `kind + settings` plus the session-only `detached_ops` marker.
2. "Linked operations inherit the profile" means the **swatch colour only**. Settings are
   a snapshot applied at link time; never live-inherited from app preferences (the
   document must stay portable).
3. An unlinked operation keeps its own colour at **50% opacity**; do not assign a new
   colour on unlink.
4. Applying a profile to operations whose kind/settings differ asks first:
   "N operations differ. Link anyway?" (Cancel / Link anyway).
5. Footer actions are icon buttons with semantic names: eyedropper = Extract,
   arrow-square-out = Export, arrow-square-in = Import. The icon set has no
   `arrow-square-in`; without a pin bump use `Icon::FolderOpen` for Import and
   `Icon::Open` for Export, with tooltips. Record the substitution in `GAPS.md`.
6. Member drag: normal drag **moves** the member to the target operation, Shift-drag
   **adds** it (copy). While Shift is held during the drag, set `CursorIcon::Copy`.
7. Palette: **10 px cells with a 1 px gap** (8 columns) plus a 1 px hover/selected ring so
   adjacent colours do not blend.
8. Profile rows use a **separate description column** and a left link indicator, dimmed,
   showing the number of operations in the current document linked to that profile.

## Photoshop study

Photoshop's chain glyph sits **between** the layer thumbnail and the linked mask
thumbnail, and clicking it breaks the link. craft-libs already models this:
`ListRow::thumb` + `extra_thumb` + `thumb_link(linked, name)` renders a dimmed link glyph
before the extra thumbnail, is a Tab stop with a focus ring, and reports
`RowEvent::LinkToggled`. Use it for operation rows: `[kind thumb][chain][profile swatch]`.

For profile rows the chain is a **read-only indicator** (chain + N), not a toggle:
clicking it would break N links at once. Paint it in the leading thumb.

## Consequences / limitations

- Links do not persist across sessions or documents (derived); "break link" is a session
  detach (`StencilApp.detached_ops`).
- A linked operation whose settings are edited loses the link automatically; there is no
  "linked but modified" state.
- `Profile.id` is for dedupe/import and future persistence; links do not reference it yet.
- Import/export use the desktop file dialog and are not covered by the harness specs.

## Acceptance

- Operations rows show `[kind thumb][chain][profile swatch]`; the chain is dimmed when
  linked, breaks on click, and is absent when unlinked.
- Linked operations show the profile colour; unlinked ones show their own colour at 50%.
- Dragging a member row onto another operation moves it; Shift-drag adds it; the cursor
  is a plus while Shift is held; the move is one undo step.
- The palette is 10 px / 1 px gaps with a visible hover ring and still exposes all 40
  swatches.
- Profile rows: link+count column, swatch, name button, description column, cog; footer
  Extract/Import/Export with icons.
- Applying a profile to differing operations asks first.
- Existing 42 baseline scenarios stay green; new scenarios are RED first, then green.
