# Deferred Items — Phase 02 (02-02 execution)

Out-of-scope discoveries logged during 02-02 execution (scope boundary: failures in
files this plan did not change, left by 02-01's uncommitted library migration):

- `crates/import/tests/corpus.rs::dxf_round_trips_our_exporter` — imported DXF shapes
  do not end with `ClosePath`. Touches `stencilcraft-dxf` export / `stencilcraft-import`
  import semantics (02-01 territory), not engine/UI code.
- `crates/import/tests/corpus.rs::svg_registry_preserves_clipped_cut_lines_and_stroke_layers` —
  imported SVG layer lookup fails. Same boundary as above.

Both fail identically without any 02-02 engine/app/UI change involved. Candidate for a
follow-up import/dxf fix wave.
