# Changelog

All notable changes to plot-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `plotdraw` — `render(f) -> Result<[PlotDrawOp], PlotFault>` and the
  twelve-operation vocabulary it produces. The package's load-bearing
  decision; the README carries the argument in full.
- `plotaxis` — `PlotScale`, `PlotTickRule`, `PlotNumberFormat`,
  `PlotTick`, `PlotAxis`, and the tick placement. Heckbert's 1-2-5 rule
  is the default and the suite's cases are his own worked examples.
- `plotseries` — line, scatter, bar and histogram, with `of_ndfloat`,
  `of_column`, `of_column_pair` and `of_table_columns` as the adapters.
  Binning rules are numpy's under their own names.
- `plotfigure` — `PlotFigure` and `PlotAxes`, placed by fractions of the
  figure, with `grid` as the whole of the layout this package does.
- `plotsvg` — the first backend: twelve match arms into `svgdoc` nodes,
  with `PlotBeginGroup`'s id passed through as an SVG `<g id>`.
- `plotraster` — the second: the same twelve into an `image-nv` `Image`,
  with `scanline_coverage` as the seam `raster-nv` would replace.
- `plotfault` — five variants, and the shortness is the design: almost
  nothing about a plot is an error.

### Decided

- **Rendering produces a value, not callbacks.** `std.canvas` is `[io]`,
  so a package that drew on one would be `host`. Splitting the package
  and an effect-parameterised sink trait were both available and are
  both argued in the README; the op list keeps the budget AND makes the
  rendering inspectable, replayable and extensible to a fourth backend
  that costs this package nothing.
- **An axes is placed by fractions, not by a rectangle.** There is no
  layout engine, because measuring a tick label needs a font file, which
  needs a filesystem. The upside is a figure that is
  resolution-independent by construction.
- **A null is a gap, not a zero**, and `of_column_pair` exists so that
  pairing two columns with nulls in different places cannot silently
  shift every point after the first gap.
- **Tick placement is a named value.** Talbot-Lin-Hanrahan is recorded
  as the variant to add, which is why `PlotTickRule` is an enum.

### Known

- `plotraster` has no rasteriser behind it yet. `raster-nv` is the plan's
  row for that work; when it lands this module becomes a caller of it
  with no change to these signatures.
- Text cannot be measured, so `plotdraw.op_bounds` on a `PlotDrawText`
  is an estimate — the same rule and the same caveat as
  `svgdoc.text_advance_estimate`.
- `geometry-nv` and `svg-nv` are PATH dependencies while the three are
  developed together. They convert to `^0.0.1` before publish, and
  publish in that order.
