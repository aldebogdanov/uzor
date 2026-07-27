# uzor

Terminal UI toolkit for [Irij](https://irij.online). Cell buffers, layout,
widgets — built on `std.term`'s `Term` effect, with the whole rendering path
kept pure so a TUI can be tested without a screen.

**v0.1 is the buffer layer only** (this PR). Layout, widgets and the app loop
land next.

```irij
use uzor.core :open
use uzor.style :open

fn frame :: Map
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
| `buffer` | `Int Int Map` | blank buffer of `cols` × `rows` |
| `buf-line` | `Map Int Vec` | one row's spans; empty outside the buffer |
| `buf-put` | `Map Int Int Vec Map` | write spans at row/col, clipped |
| `buf-text` | `Map Int Int Map Str Map` | write a styled string |
| `buf-fill` | `Map Map Map Str Map` | fill a rect with a repeated character |

Writes are clipped at the buffer's edges and rows outside it are dropped, so
a widget that miscomputes its own size corrupts its own region and never the
rest of the frame. A negative column clips the front of the content rather
than shifting it.

### Lines and spans

A line is a vector of **spans** — `{text= "hi" st= <style>}` — not a vector of
cells. A 200×50 screen is 10 000 cells and Irij's vectors are persistent, so
building one cell at a time is quadratic; a line usually holds a handful of
spans instead. `line-splice` still gives exact cell-level placement:

| fn | spec | |
|---|---|---|
| `span` / `span-width` | `Str Map Map` / `Map Int` | construct, measure in cells |
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
| `buf-render` | `Map Str` | clear screen, then every row |
| `buf-diff` | `Map Map Str` | bytes carrying `prev` → `next` |
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

## Styles

A style is a map; `style {fg= red bold= true}` fills the rest from `plain`.

| | |
|---|---|
| colours | `Int` — `-1` is the terminal default, `0..255` the xterm palette |
| names | `black` … `white`, `bright-black` … `bright-white` |
| `grey n` | grey ramp, 0–23 |
| `rgb6 r g b` | 6×6×6 cube, components 0–5 |
| `with-fg` / `with-bg` | `Map Int Map` |
| `sgr` | `Map Str` — the escape that puts a terminal into this style |

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

## Known constraint

Internal lambdas avoid capturing their enclosing function's parameters, using
explicit recursion instead. A captured parameter currently resolves to a
same-named top-level binding in the *calling* program — an Irij scoping bug
that made `sgr` return the wrong style for any app with a global named `st`.
`tests/test-render.irj` pins it.

## Development

```
irij test        # 52 tests, no terminal required
```

## License

MIT
