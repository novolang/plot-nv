# plot-nv

A chart is a picture of numbers: a pair of axes, a scale on each, and
one or more series of data drawn inside them. This package builds that
picture in novo-lang and answers it as a list of primitive drawing
operations, which an SVG document, a raster image or a desktop canvas
each replay. Its references are the Rust crate
[plotters](https://docs.rs/plotters) for the split between a figure and
its backends and
[matplotlib](https://matplotlib.org/stable/api/index.html) for the names
of the operations. It reads arrays from
[ndarray-nv](https://novo-lang.org/packages/ndarray-nv) and columns from
[dataframe-nv](https://novo-lang.org/packages/dataframe-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the pieces are

A **figure** is the whole drawing: a size, a background, an optional
title, and one or more sets of **axes**. A set of axes is a pair of
axes, the **series** drawn inside them, a title and a legend. A
**series** is one run of data with a label and a style.

An **axis** carries a **scale**, a range, a tick rule, a number format
and whether to draw grid lines. A **scale** maps a data value to a
position between 0 and 1 along the axis. Three are available: linear,
logarithmic, and **symmetric logarithmic**, which is logarithmic away
from zero and linear near it, so that data crossing zero can still be
drawn on a log-like axis.

A **tick** is one labelled position on an axis. Choosing which positions
to label is a real decision: 0, 2.5, 5, 7.5, 10 reads well and 0, 2.857,
5.714 does not. A **tick rule** is that decision as a value.

A **bin** is one bar of a histogram. A **binning rule** decides how many
bins and where their edges fall. The same data under two rules can look
as if it has one peak or two.

**Figure coordinates** are the coordinates every drawing operation is
in. The origin is the top left, `y` grows downwards, and the units are
the figure's own. They are not data coordinates, because the scales have
already been applied, and not pixels, because the figure does not know
how large the drawing will be.

A **draw operation** is one primitive instruction: fill this rectangle,
stroke this polyline, place this text. There are twelve, and a backend
that handles twelve is finished.

## Install

```
novo pkg add plot-nv
```

## Example

```novo
use std.list
use plotseries
use plotfigure
use plotdraw
use plotsvg
use svgwrite

fn main() [io]
    // A line through five points, labelled for the legend.
    let s = plotseries.line("temperature", [0.0, 1.0, 2.0, 3.0, 4.0],
                                           [3.1, 3.6, 4.0, 3.8, 4.4])

    // One set of axes filling a 640-by-400 figure, with a title.
    let fig = plotfigure.with_title(plotfigure.single(640.0, 400.0, [s]), "measurements")

    // Rendering answers a list of primitive draw operations. Nothing is
    // drawn and nothing is written.
    match plotdraw.render(fig)
        Err(e)   => println(e.message())
        Ok(ops)  => println("${list.len(ops)} draw operations")

    // One backend replays that list as an SVG document.
    match plotsvg.to_svg_string(fig, svgwrite.minimal())
        Err(e)   => println(e.message())
        Ok(text) => println("${text.len()} characters of SVG")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: plot-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `plotseries` | The four series kinds, their constructors and styles, the seven marker shapes, the four binning rules, the conversions from an array and from a dataframe column, the extents, and the default palette. |
| `plotaxis` | The three scales, the five tick rules, the five number formats, an axis and its options, tick placement, the mapping between a value and a position, and the rounding rule underneath tick placement. |
| `plotfigure` | The figure, a set of axes, their placement as fractions of the figure, a regular grid of them, titles, backgrounds, legends, gutters, the rectangles they resolve to, and validation. |
| `plotdraw` | The twelve draw operations, the renderer, the per-axes and per-series renderers, the queries over an operation list, and the two geometry helpers a backend needs. |
| `plotsvg` | Turning a figure or an operation list into an SVG document or into SVG text. |
| `plotraster` | Turning a figure or an operation list into an image, compositing one onto an image the caller owns, the pixel size arithmetic, and the scanline coverage and blending underneath. |
| `plotfault` | Every reason a figure refuses to render, as one enum with five variants. |

## How to choose an entry point

**`plotfigure.single` is the one-axes case** and `plotfigure.figure`
takes several. `plotfigure.grid` computes the placements for a regular
grid of subplots.

**`plotdraw.render` is the only renderer.** Everything downstream reads
its list.

| Backend | Call | Where it runs |
| --- | --- | --- |
| SVG | `plotsvg.to_svg` or `.to_svg_string` | here, and it performs nothing |
| Raster | `plotraster.to_image` | here, and it performs nothing |
| A desktop canvas | in the caller | wherever the canvas is |

The third one is a loop over the twelve operations in the caller's own
code. This package never mentions a canvas, which is what keeps it free
of any effect and usable from a program that only wants an SVG file.
`plotdraw.circle_as_polygon` and `plotdraw.dash_polyline` are published
for that loop, because a canvas has paths and no circles and no dashes.

**Ask questions of the operation list rather than of a picture.**
`plotdraw.op_count` says how many operations a figure will produce,
`ops_bounds` gives the rectangle they cover, and `ops_in_group` selects
the ones a named part of the chart produced. A test can assert that a
chart contains eleven rectangles without drawing anything.

**Render once and replay twice.** The same list goes to a file and to a
window.

## The rules a user needs

1. **Rendering answers a value, and nothing here draws.** No function in
   this package performs any input or output.
2. **Draw operations are in figure coordinates**, with the origin at the
   top left and `y` growing downwards. `plotraster` multiplies by its
   scale and `plotsvg` writes them as user units under a matching view
   box.
3. **The operation vocabulary is primitive.** Markers are expanded into
   circles and polygons by the renderer, a dash is a list of lengths on
   the operation rather than a style object, and text is a placed run
   with no layout.
4. **The default tick rule is Heckbert's.** Take the range, divide by
   the number of ticks wanted, round the spacing up to the nearest 1, 2,
   5 or 10 times a power of ten, and place ticks at multiples of it. It
   is the loose-label algorithm of "Nice Numbers for Graph Labels",
   Graphics Gems I, 1990. `plotaxis.nice_number` is the rounding step on
   its own.
5. **A tick rule is a value you can name.** `PlotTicksNice` is
   Heckbert's, `PlotTicksCount` asks for a number of them,
   `PlotTicksStep` fixes the spacing and the origin, `PlotTicksAt` lists
   them, and `PlotTicksDecade` places one per power of the base.
6. **A binning rule is a value you can name too.** `PlotBinsSturges` and
   `PlotBinsFreedmanDiaconis` are NumPy's two rules,
   `PlotBinsCount` fixes the number and `PlotBinsWidth` the width. The
   same data under two rules can look unimodal or bimodal, so the
   picture says which rule produced it.
7. **A null in a dataframe column is a gap, not a zero.**
   `plotseries.of_column` drops the absent positions, and a line drawn
   from the result is broken there rather than joined across it. A line
   that bridges a gap asserts data the frame does not have. This follows
   matplotlib's handling of a masked array.
8. **Pair two columns with `of_column_pair`, never with two
   `of_column` calls.** The pair drops a row when either side is absent
   and keeps the two lists aligned. Two separate calls on columns whose
   nulls fall in different places give lists of different lengths and
   shift every point after the first gap.
9. **An axes is placed as fractions of the figure**, from 0 to 1, as
   matplotlib's `add_axes` rectangle is. There is no layout engine,
   because measuring a tick label needs a font file and this package
   reads nothing. The consequence is that one figure value renders at
   640 units and at 1920 with every axes in the same relative place.
10. **Gutters are fractions the caller sets.** They are the space
    reserved for tick labels, axis labels and a title.
11. **A logarithmic axis refuses a range that includes zero or a
    negative number.** `PlotLogRangeInvalid` names the axis and the
    range, and `plotaxis.range_is_valid` asks in advance. Use a
    symmetric logarithmic scale for data that crosses zero.
12. **A series whose two lists differ in length is refused.**
    `PlotLengthMismatch` names the series and both lengths, and
    `plotseries.is_consistent` asks in advance.
13. **The operation list carries no type from any backend.**
    `PlotAnchor` is this package's own rather than svg-nv's text anchor,
    so a caller replaying the list onto a canvas does not take svg-nv
    into its dependency closure to read one enum.

## What is not included

- **A layout engine.** See rule 9.
- **Box plots, violin plots, heat maps, contour plots, error bars,
  stacked areas and pie charts.** The four series kinds here are the
  ones a notebook corpus actually contains. Each of the others is a
  different layout problem rather than a variation on these.
- **The extended Wilkinson tick algorithm** of Talbot, Lin and Hanrahan,
  which scores candidate tick sets and produces better axes on awkward
  ranges at the cost of a search. `PlotTickRule` is an enum so that it
  can arrive as another variant.
- **Text metrics and font loading.** See rule 9. `PlotDrawText` carries
  a size and an anchor, and the backend measures.
- **Interactivity, animation and three-dimensional axes.**
- **A microcontroller build.** A figure and its operation list are
  growable lists. This package makes no device claim and ships no device
  probe.

## Related packages

- [geometry-nv](https://novo-lang.org/packages/geometry-nv) owns the
  points, rectangles and transforms a figure is laid out with.
- [svg-nv](https://novo-lang.org/packages/svg-nv) is the document the
  first backend writes.
- [image-nv](https://novo-lang.org/packages/image-nv) is the raster the
  second backend fills.
- [color-nv](https://novo-lang.org/packages/color-nv) owns every colour
  in a figure and in a draw operation.
- [ndarray-nv](https://novo-lang.org/packages/ndarray-nv) and
  [dataframe-nv](https://novo-lang.org/packages/dataframe-nv) are the
  input. `plotseries.of_ndfloat` and `.of_column` are the two doors.
- [stats-nv](https://novo-lang.org/packages/stats-nv) computes the
  summaries a chart is usually drawn beside.

## Tests

```bash
novo test tests/plot_tests.nv          # 33 tests: ticks, binning, series and the op list
```

plotters is the reference for the backend split and matplotlib for the
operations. The tick cases are Heckbert's own worked examples from
Graphics Gems I, and the binning cases are NumPy's documented rules.

The suite asserts that a figure renders to the operations it should
without anything being drawn, that a null in a column breaks a line
rather than joining across it, that `of_column_pair` keeps two columns
aligned where two separate calls would not, that a logarithmic axis
refuses a range containing zero, that a series with mismatched lengths
is refused, and that an axes placed by fractions lands in the same
relative place at two figure sizes.

The tests compile today and fail at run, each on the
`not implemented: plot-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `plotseries.PlotSeries`, `.PlotSeriesKind`, `.PlotMark`, `.PlotOrientation`, `.PlotBinRule` | declared |
| `plotaxis.PlotAxis`, `.PlotTick`, `.PlotScale`, `.PlotTickRule`, `.PlotNumberFormat` | declared |
| `plotfigure.PlotFigure`, `.PlotAxes`, `.PlotLegend`, `.PlotLegendPos` | declared |
| `plotdraw.PlotDrawOp`, `.PlotAnchor`, `.PlotRun`, `plotfault.PlotFault` | declared |
| `plotseries.line`, `.scatter`, `.bars`, `.histogram` | no |
| `plotseries.with_color`, `.with_mark`, `.with_line`, `.with_alpha`, `.default_style`, `.palette_color` | no |
| `plotseries.of_ndfloat`, `.of_column`, `.of_column_pair`, `.of_table_columns` | no |
| `plotseries.point_count`, `.is_consistent`, `.x_extent`, `.y_extent`, `.mark_name`, `.kind_name` | no |
| `plotseries.bin_edges`, `.bin_counts` | no |
| `plotaxis.linear`, `.log10`, `.categorical`, and the five `with_` options | no |
| `plotaxis.is_auto_range`, `.padded_range`, `.ticks_for`, `.ticks_of`, `.tick`, `.nice_number` | no |
| `plotaxis.position`, `.value_at`, `.range_is_valid`, `.format_value`, `.auto_format` | no |
| `plotfigure.figure`, `.single`, `.axes`, `.grid`, `.add_axes`, `.add_series` | no |
| `plotfigure.with_title`, `.with_background`, `.with_axes_title`, `.with_legend`, `.with_gutters`, `.placed_at` | no |
| `plotfigure.legend_at`, `.no_legend`, `.box_of`, `.plot_area_of`, `.data_to_figure`, `.resolved`, `.validate` | no |
| `plotdraw.render`, `.render_axes`, `.render_series`, `.op_count` | no |
| `plotdraw.op_name`, `.op_bounds`, `.ops_bounds`, `.ops_in_group`, `.group_ids` | no |
| `plotdraw.circle_as_polygon`, `.dash_polyline` | no |
| `plotsvg.to_svg`, `.to_svg_string`, `.ops_to_svg`, `.ops_to_svg_with_font`, `.op_to_nodes`, `.anchor_to_svg` | no |
| `plotraster.to_image`, `.ops_to_image`, `.ops_onto`, `.pixel_size`, `.scanline_coverage`, `.blend` | no |
| `plotfault`'s five variants, `.summary` and its `Error` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
