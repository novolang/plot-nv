# plot-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The plotting subset a notebook uses: a figure with axes, line, scatter,
bar and histogram series over `ndarray-nv` arrays and `dataframe-nv`
columns, tick placement as a named algorithm, legends and labels — and
**rendering as a pure function into a list of draw operations** that an
SVG document, a raster image or a desktop canvas each replay.

It is the chart in [`orbit/novobook`](../novobook), whose README says
today that it has no plots.

```
novo pkg add plot-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use plotseries
use plotfigure
use plotsvg
use svgwrite

// A dataframe's columns as a chart, as SVG text a notebook can put in
// its output cell.
fn chart_of(t: DfTable, x: Str, ys: [Str]) -> Result<Str, PlotFault>
    let fig = plotfigure.single(640.0, 400.0, plotseries.of_table_columns(t, x, ys))
    plotsvg.to_svg_string(plotfigure.with_title(fig, "measurements"), svgwrite.minimal())
```

## The layer, and why — the load-bearing decision

`core`, and the row asked for something that makes that hard. Three
targets: an SVG document, a raster, and **`std.canvas` draw calls for the
desktop**. The third is the problem.

`std.canvas`'s entry points are `[io]` — `novo_canvas_draw_rects`,
`novo_canvas_path_fill_solid` and the rest all declare it. A function
here that called one would inherit `[io]`, leave `core`'s empty budget,
and make plot-nv a `host` package. That is not a formality: a `host`
plot-nv cannot be depended on by anything in `core`, cannot build for
wasm the way a notebook needs, and drags a window system into the
dependency closure of a program that only wanted a PNG.

Three ways out were available.

| | cost |
| --- | --- |
| **Split the package** — plot-core-nv and plot-canvas-nv, the way calendar/chrono and config-core/config split | a whole extra registry row, a second README and a version to keep in step, for one function |
| **A trait with an effect parameter** — `trait PlotSink[e]`, `render<S: PlotSink[e]>(f, s) -> [e]` (SPEC §5.6) | the renderer becomes a callback-driven traversal: the caller cannot pause it, inspect what is about to be drawn, count operations, or render once and replay twice — and every backend must implement a trait to receive anything, including the two that just want a value |
| **Render to a value** — `render(fig) -> [PlotDrawOp]` | one intermediate allocation, and an op vocabulary that has to be complete enough |

**The third is chosen**, and what it buys beyond the budget is the
reason it would be right anyway:

- **Two of the three targets are then in this package and pure.**
  `plotsvg` turns ops into an `SvgDocument`; `plotraster` turns them into
  an `image-nv` `Image`. Neither performs anything.
- **The third is thirty lines in the caller** — see below. plot-nv never
  mentions `std.canvas`, which is exactly why it stays `core`.
- **The op list is inspectable.** A test asserts that a chart contains
  eleven rectangles without rasterising anything. A notebook counts
  operations before deciding whether to send the SVG or a PNG. A caller
  draws the chart twice — once to a canvas, once to a file — from one
  render.
- **A fourth backend costs this package nothing.** A terminal plot over
  `tui-nv`, a plotter's G-code, a PDF: each is a reader of the same list,
  written where it belongs.

### The third backend, in full

This is the whole of what a `host` program writes to put a chart on a
desktop canvas. It lives in the caller, not here.

```novo ignore
fn replay(c: Canvas, ops: [PlotDrawOp]) [io]
    for op in ops
        match op
            PlotFillRect(x, y, w, h, col) =>
                c.rect(x, y, w, h, col)
            PlotStrokeRect(x, y, w, h, col, lw) =>
                c.path([(x, y), (x + w, y), (x + w, y + h), (x, y + h)], true)
                c.stroke(lw, col)
            PlotStrokeLine(x1, y1, x2, y2, col, lw, dash) =>
                for run in plotdraw.dash_polyline([pt(x1, y1), pt(x2, y2)], dash, 0.0)
                    c.path(run.points, false)
                    c.stroke(lw, col)
            PlotStrokePolyline(pts, col, lw, dash) =>
                for run in plotdraw.dash_polyline(pts, dash, 0.0)
                    c.path(run.points, false)
                    c.stroke(lw, col)
            PlotFillPolygon(pts, col) =>
                c.path(pts, true)
                c.fill(col)
            PlotFillCircle(cx, cy, r, col) =>
                c.path(plotdraw.circle_as_polygon(cx, cy, r, 16), true)
                c.fill(col)
            PlotStrokeCircle(cx, cy, r, col, lw) =>
                c.path(plotdraw.circle_as_polygon(cx, cy, r, 16), true)
                c.stroke(lw, col)
            PlotDrawText(s, x, y, size, col, anchor, rot) =>
                c.text(s, x, y, size, col, anchor, rot)
            PlotSetClip(x, y, w, h) => c.clip(x, y, w, h)
            PlotClearClip           => c.clip_none()
            PlotBeginGroup(_)       => ()
            PlotEndGroup            => ()
```

`circle_as_polygon` and `dash_polyline` are in `plotdraw` for exactly
this: `std.canvas` has paths and no circle and no dashes, and the
trigonometry and the phase arithmetic belong in one place rather than in
every backend.

## The load-bearing interface

```novo ignore
pub fn render(f: PlotFigure) -> Result<[PlotDrawOp], PlotFault>   // the only renderer

pub enum PlotDrawOp          // twelve operations, all primitive
    PlotBeginGroup(id: Str)  PlotEndGroup
    PlotSetClip(…)           PlotClearClip
    PlotFillRect(…)          PlotStrokeRect(…)
    PlotStrokeLine(…)        PlotStrokePolyline(points: [GeomPointF], …)
    PlotFillPolygon(…)       PlotFillCircle(…)  PlotStrokeCircle(…)
    PlotDrawText(text: Str, x: Float, y: Float, size: Float, …)
```

The vocabulary is deliberately primitive: markers are expanded into
circles and polygons by the renderer, a dash is a pattern on the op
rather than a style object, and there is no text layout — only a placed
run. **A backend that handles twelve operations is done.**

Coordinates are **figure coordinates**: y grows down from the top left,
in the figure's own units. Not data coordinates — the scales have
already been applied — and not pixels, because the figure does not know
how big the drawing is. `plotraster` multiplies by its scale; `plotsvg`
writes them as user units under a matching `viewBox`.

`PlotAnchor` is this package's own rather than svg-nv's
`SvgTextAnchor`, and `PlotRun` rather than `SvgSubpath`, for one reason:
an op list must not carry a type from one of its three backends, or the
canvas replay — which wants nothing to do with SVG — would take svg-nv
into its dependency closure to read an enum.

## Tick placement is a named algorithm

"Pick some round numbers" is where every plotting library quietly
differs from every other, and it is the first thing a reader of a chart
notices: labels at 0, 2.5, 5, 7.5, 10 read well and labels at 0, 2.857,
5.714 do not. So the rule is a **value** — `PlotTickRule` — that a caller
can name, compare and test.

The default is **Heckbert's**: the loose-label algorithm from "Nice
Numbers for Graph Labels" (Graphics Gems I, 1990). Take the range,
divide by the number of ticks wanted, round that spacing up to the
nearest 1, 2, 5 or 10 times a power of ten, place ticks at multiples of
it. Twenty lines, no tables, and the answer is a number a person would
have chosen — it is what gnuplot, plotters and matplotlib's
`MaxNLocator` each do a version of. The suite's cases are Heckbert's own
worked examples.

**Talbot, Lin and Hanrahan's extended Wilkinson algorithm is deliberately
not here.** It scores candidate tick sets on simplicity, coverage,
density and legibility and produces better axes on awkward ranges, at
the cost of a search. It belongs here eventually, as a fifth variant
named after it — which is why `PlotTickRule` is an enum and not a
boolean.

Histogram binning is the same shape: `PlotBinsSturges` and
`PlotBinsFreedmanDiaconis` are numpy's rules under their own names,
because the same data binned by the two can look unimodal or bimodal,
and a caller should be able to say which picture they are looking at.

## What it ports

[plotters](https://github.com/plotters-rs/plotters) for the backend
split and the series set, and the
[matplotlib](https://matplotlib.org/) subset a notebook corpus actually
contains for the API. Four series kinds — line, scatter, bar, histogram
— is a measurement, not a shortlist: the long tail after them (box,
violin, heatmap, contour, error bars, stacked areas, pie) is each a
different layout problem rather than a variation on these, and each is a
variant here plus a branch in `plotdraw` when someone needs it.

## Two things this package does not do, and says so

**There is no layout engine.** matplotlib's `tight_layout` measures every
tick label and title and solves for margins that fit them — which needs
text metrics, which need a font file, which needs a filesystem. This
package is `core` and has none. So an axes is placed by **fractions** of
the figure (matplotlib's `add_axes` rectangle), `plotfigure.grid`
computes those fractions for a regular grid of subplots, and the gutters
are fractions a caller can set from its own measurements. The upside is
that a figure is resolution-independent by construction: the same value
renders at 640 pixels and at 1920 with every axes in the same relative
place.

**There is no rasteriser yet.** `plotraster`'s target is a scanline
rasteriser with analytic coverage — stb_truetype's and tiny-skia's
approach, which antialiases without supersampling. `raster-nv` is the
plan's row for that work as a package of its own (P2); until it lands
this module carries it, and when it lands this module becomes a caller
of it **with no change to the interface below**.
`plotraster.scanline_coverage` is the seam, and it is public because it
is the thing to test.

## A null is a gap, not a zero

`plotseries.of_column` drops the positions a dataframe column marks
absent, and a line series drawn from one is broken there rather than
joined across it — because a line that bridges a gap asserts data that is
not in the frame. That is matplotlib's behaviour for a masked array and
pandas's for a NaN.

The consequence is that the result can be shorter than the column, so
pairing two columns uses `of_column_pair`, which drops a row when
**either** side is null and keeps the two aligned. Two separate
`of_column` calls on columns with nulls in different places give lists of
different lengths and silently shift every point after the first gap —
which is the bug that function exists to make unavailable.

## Related

- [`geometry-nv`](https://github.com/novolang/geometry-nv) — the
  rectangles and transforms a figure is laid out with
- [`svg-nv`](https://github.com/novolang/svg-nv) — the first backend
- [`ndarray-nv`](https://github.com/novolang/ndarray-nv) and
  [`dataframe-nv`](https://github.com/novolang/dataframe-nv) — the input
- [`image-nv`](https://github.com/novolang/image-nv) — the raster target
- [Publishing a package to Orbit](https://novo-lang.org/publishing) —
  the layer rules this package is held to
