# uzor

Terminal UI toolkit for [Irij](https://irij.online). Cell buffers, layout,
widgets — built on `std.term`'s `Term` effect, with the whole rendering path
kept pure so a TUI can be tested without a screen.

**v0.1 is the buffer and layout layers.** Widgets and the app loop land next.

```irij
use uzor.core :open
use uzor.style :open

fn frame :: Buffer
  =>
  buf := buffer 40 5
  titled := buf-text buf 0 2 (style {bold= true fg= cyan}) "uzor"
  buf-text titled 2 2 plain "a frame is a value"

;; What the terminal would receive, as a plain string:
buf-render (frame ())
```

## Install

```toml
[seeds]
uzor = "0.1"
```

## Why values

Drawing is a pure function from frame to frame. `buf-diff` turns two frames
into the bytes that carry the terminal from one to the other. Only the app
loop needs a terminal — so every widget above this layer is tested by
asserting on a string, with no tty, no mock and no handler.

## Buffers

| fn | spec | |
|---|---|---|
| `buffer` | `Int Int Buffer` | blank buffer of `cols` × `rows` |
| `buf-line` | `Buffer Int Vec` | one row's spans; empty outside the buffer |
| `buf-put` | `Buffer Int Int Vec Buffer` | write spans at row/col, clipped |
| `buf-text` | `Buffer Int Int Style Str Buffer` | write a styled string |
| `buf-fill` | `Buffer Rect Style Str Buffer` | fill a rect with a repeated character |

Writes are clipped at the buffer's edges and rows outside it are dropped, so
a widget that miscomputes its own size corrupts its own region and never the
rest of the frame. A negative column clips the front of the content rather
than shifting it.

### Lines and spans

A line is a vector of **spans** — `Span "hi" <style>` — not a vector of
cells. A 200×50 screen is 10 000 cells and Irij's vectors are persistent, so
building one cell at a time is quadratic; a line usually holds a handful of
spans instead. `line-splice` still gives exact cell-level placement:

| fn | spec | |
|---|---|---|
| `span` / `span-width` | `Str Style Span` / `Span Int` | construct, measure in cells |
| `line-width` | `Vec Int` | total cells |
| `line-take` | `Vec Int Vec` | first `n` cells, **padded** with blanks if short |
| `line-drop` | `Vec Int Vec` | everything after the first `n` cells |
| `line-splice` | `Vec Int Vec Vec` | replace `[col, col+width)` — every draw goes through this |

A double-width character cut in half becomes a space, on both sides of the
cut. Letting it run one cell long would shift everything after it.

## Rendering

| fn | spec | |
|---|---|---|
| `render-line` | `Vec Str` | spans → bytes; empty line → `""` |
| `buf-render` | `Buffer Str` | clear screen, then every row |
| `buf-diff` | `Buffer Buffer Str` | bytes carrying `prev` → `next` |
| `move-to` | `Int Int Str` | 1-based cursor addressing |

The diff is per line: a changed line is rewritten whole (cursor move, content,
erase-to-end-of-line) rather than patched column by column. Terminal bandwidth
is nowhere near the bottleneck, and a whole-line rewrite can't desynchronise
from what's on screen. **Identical frames diff to the empty string**, so an
idle app is silent. A size change repaints everything — after a reflow every
line below the first has moved.

Each span carries its full SGR state followed by a reset. A differential
encoder would have to know what the terminal is already showing, and any drift
smears colour across the screen.

## Layout

Splitting a region into sub-regions, one axis at a time. Nesting the two
directions covers every layout the widget set needs, and the whole thing stays
a pure function from a rect and a list of constraints to a list of rects — no
solver state, no back-tracking, no invalidation.

| constraint | |
|---|---|
| `Fixed n` | exactly `n` cells |
| `Percent n` | `n`% of the region, rounded down |
| `Fill w` | share of what's left, weighted |
| `Min n` | at least `n`, then shares like a fill |
| `Max n` | shares like a fill, but stops at `n` |

| fn | spec | |
|---|---|---|
| `solve` | `Int Vec Vec` | sizes along one axis |
| `split-h` / `split-v` | `Rect Vec Vec` | left-to-right / top-to-bottom |
| `grid` | `Rect Vec Vec Vec` | rows of columns; a cell is `nth col (nth row g)` |
| `center` | `Rect Int Int Rect` | a `w`×`h` region centred, clamped to the parent |
| `pad` | `Rect Int Int Int Int Rect` | uneven inset, CSS order (top, right, bottom, left) |
| `inner` | `Rect Rect` | one-cell inset — the inside of a border |

```irij
panes := split-h (rect 0 0 80 24) #[(Fixed 20) (Fill 1)]
rows  := split-v (nth 1 panes) #[(Fixed 1) (Fill 1) (Fixed 1)]
```

**Sizes sum to exactly the region.** The last flexible child absorbs the
rounding, so a three-way split of 100 comes out 33/33/34 rather than losing a
cell — a lost cell is a column of stale content that nothing ever redraws.
Over-subscription (a fixed sidebar in a narrow terminal) clamps: earlier
children win, later ones collapse to zero, and nothing ever goes negative.

## Borders

`uzor.box` draws frames — `single-box`, `rounded-box`, `double-box`,
`heavy-box`, and `ascii-box` for a terminal that mangles the box-drawing
block. Same shape, so swapping the set changes nothing else.

| fn | spec | |
|---|---|---|
| `draw-box` | `Buffer Rect BoxChars Style Buffer` | a frame; interior untouched |
| `draw-titled-box` | `Buffer Rect BoxChars Style Str Buffer` | title let into the top edge |
| `draw-panel` | `Buffer Rect BoxChars Style Buffer` | frame plus a cleared interior |

A box narrower or shorter than two cells has no inside, so it draws nothing
rather than smearing its two edges into one column. Titles truncate by cells,
so a CJK title can't measure a cell over budget and overwrite the corner. Use
`draw-panel` for anything floating — a modal that only drew its border would
show the old frame through its middle.

## Styles

`style {fg= red bold= true}` fills the rest from the defaults; the result is
a `Style` value, reached by `.fg`, `.bold` and so on.

| | |
|---|---|
| colours | `Int` — `-1` is the terminal default, `0..255` the xterm palette |
| names | `black` … `white`, `bright-black` … `bright-white` |
| `grey n` | grey ramp, 0–23 |
| `rgb6 r g b` | 6×6×6 cube, components 0–5 |
| `with-fg` / `with-bg` | `Style Int Style` |
| `sgr` | `Style Str` — the escape that puts a terminal into this style |

The 256-colour palette is used rather than truecolour because it degrades
gracefully: a 256-colour escape renders somewhere sane on everything from tmux
to a bare Linux console, while a truecolour one can come out as noise.

### Width

`cell-width` measures cells, not characters — a CJK ideograph is two columns
and a combining mark none, and laying out by `length` drifts on both. This
duplicates what `term-str-width` gets from JLine, deliberately: the buffer
layer stays pure, and an app that wants the terminal's own opinion can still
ask for it.

Emoji ZWJ sequences (a flag, a family) count as their parts rather than one
cluster, so they over-count. Terminals disagree about those anyway.

## Shapes

`Rect`, `Span`, `Buffer` and `Style` are specs, not bare maps, so every fn
boundary checks what it was handed — a rect missing a field is rejected where
it enters rather than surfacing three frames later as a nonsense cursor
position. Fields are reached with `.x`, `.text`, `.cols`; updates go through
the constructor, since record-update syntax is Map-only.

A *line* is deliberately a plain `Vec` of spans. It's touched several times per
drawn string, and validating a spec per element would put a walk of the whole
line on every splice.

Product specs are closed and their field types are checked, so `Rect` means
exactly four Int fields: a wrong type, a missing field and an extra field are
all rejected where the value enters. Construction is checked too, so a bad
`Rect` is caught where it is built rather than three frames later as a nonsense
cursor position.

## Development

Needs Irij ≥ 0.8.8 — earlier builds mis-render styles when a program has a
top-level binding named `st` (irij#11), and don't check product-spec fields
(irij#12).

```
irij test        # 110 tests, no terminal required
```

## License

MIT
