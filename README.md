# uzor

Terminal UI toolkit for [Irij](https://irij.online). Cell buffers, layout,
widgets — built on `std.term`'s `Term` effect, with the whole rendering path
kept pure so a TUI can be tested without a screen.

**v0.1 is complete**: buffers, layout, widgets and the app loop. A reactive
binding on top of [butterfly](https://github.com/aldebogdanov/butterfly) is
next — see *Reactive state*, below.

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

## Text

`uzor.text` fits strings into a fixed number of cells. Everything counts cells,
never characters — padding with `length` puts a CJK label one column past its
box and shifts every column after it.

| fn | spec | |
|---|---|---|
| `truncate` | `Str Int Str` | first `n` cells; a halved wide glyph becomes a space |
| `ellipsize` | `Str Int Str` | truncate with `…`, marker inside the budget |
| `fit` | `Str Int Align Str` | pad **or** truncate to exactly `n` cells |
| `wrap` | `Str Int Vec` | word-wrap; over-long words break mid-word |

`Align` is `Left` / `Center` / `Right`.

## Widgets

Every widget is a pure `Buffer Rect <model> … Buffer`. They hold no state and
perform no effects: the model is a value the caller owns, and drawing is a
function of it. That's what lets a screenful of widgets be asserted on as a
string.

| fn | |
|---|---|
| `draw-label` | one line, aligned, ellipsized to fit |
| `draw-paragraph` | wrapped text, clipped to the rect's height |
| `draw-list` | items with a selection bar |
| `draw-table` | headers + rows; columns are layout constraints |
| `draw-input` | text with a cursor cell |
| `draw-progress` / `draw-progress-labelled` | bar from `value`/`total` |
| `draw-spinner` | braille frame from a tick |

```irij
lv := list-scroll (list-view items selected offset) r.h
buf2 := draw-list buf r lv plain (style {reverse= true})
```

**Scrolling is not hidden inside the widgets.** A list that silently adjusted
its own offset while drawing would make drawing stateful and put the app's idea
of the scroll position out of step with what it sees. Each scrollable widget has
a pure companion — `list-scroll`, `table-scroll`, `input-scroll` — that the app
calls first. `table-scroll` reserves the header row, or the last row is never
reachable.

Progress takes `value` and `total` as **integers**, not a ratio: a bar is a
whole number of cells, and going through a float only adds a rounding step whose
direction has to be argued about. A total of zero draws an empty bar rather than
dividing by it.

The input cursor is a reversed cell, not the terminal's own cursor — the frame
stays a value, so a diff of two frames still describes everything that changed.

## Events

`uzor.event` names a key with one string. `std.term` hands over
`{kind= "key" key= "c" mods= #["ctrl"]}`; matching that field by field at every
call site is noise and gets the modifier order wrong sooner or later.

| fn | |
|---|---|
| `key-desc` / `key?` | `"ctrl+c"`, and matching against it |
| `key-event?` / `resize-event?` / `mouse-event?` / `idle-event?` / `eof-event?` | kind predicates |
| `printable?` | a character that could go into a text field |
| `keymap` / `lookup` / `lookup-in` | descriptor → your own message |
| `focus` / `focus-next` / `focus-prev` / `focus-on` / `focused` / `focused?` | a ring of component ids |

Modifier order is fixed, because descriptors are compared as strings — if it
floated, `"ctrl+shift+tab"` and `"shift+ctrl+tab"` would be different keys. An
unbound key yields `()` rather than an error: most keys in most apps mean
nothing, and that is not a failure. `lookup-in` tries tables in order, so a
component's bindings can take precedence over the app's globals without either
knowing about the other.

## The app loop

`uzor.app` is the only module that touches a terminal. Everything below it is a
pure function of values; this is where those values meet the outside.

```irij
run-app (app initial-model update view done?) {alt-screen= true}
```

| | |
|---|---|
| `update` | `model event -> model` |
| `view` | `model size -> Buffer` |
| `done?` | `model -> Bool` |

`done?` rather than a quit message out of `update`: a message protocol would
have to be understood by every update fn in every app, and "has this model
finished" is a question the model can already answer.

`run-app` returns the final model. The terminal is restored on the way out
**including when the app errors** — a full-screen program that dies without
restoring leaves the user with an unusable shell.

A resize repaints from scratch rather than diffing, since after a reflow every
line below the first has moved. The event still reaches `update`, in case the
app wants to know.

`render-once` draws a model and returns the bytes with no loop and no raw mode
— for a one-shot report, or for looking at a view in isolation.

### Testing a whole app

`run-app` performs `Term` and nothing else, so an application runs under a mock
handler with scripted keys and **no terminal at all**:

```irij
handler script-term :: Term
  st :! {keys= #["down" "down" "q"]}
  term-read ms => …            ;; hand back the next key
  …

fn run
  =>
  with script-term
    run-app (app init update view done?) {}

result := run ()               ;; the model the app finished on
```

That is what the pure layers below are for. See `tests/test-app.irj`, which
also covers eof, resize, an app that starts finished, and that the terminal is
restored after an error.

## Reactive state

`uzor.app` folds every event into one model. That's the right default and most
apps should stay there. `uzor.reactive` is the alternative for apps whose state
stops fitting in one record — state lives in
[butterfly](https://github.com/aldebogdanov/butterfly) signals, and the view is
a `compute` over them.

```irij
fn build ::: Reactive
  => todos sel size-sig
  visible := compute (-> filter-todos (deref todos) …)
  view    := compute (-> draw-screen (deref size-sig) (deref visible) (deref sel))
  reactive-app view (ev -> route todos sel ev) (-> deref quit)

run-reactive (setup-of todos sel) {alt-screen= true}
```

`setup` is handed a signal carrying the terminal size and returns three things:
the `view` compute, a `dispatch` fn, and `done?`. `run-reactive` returns a
snapshot of the final graph.

What it buys:

- **Composable state.** Each part of the UI owns its signals. Adding a pane
  doesn't touch the others, and nothing threads a growing record through every
  update fn.
- **Derived values that recompute themselves.** A `compute` re-runs when its
  inputs change and is cached when they don't. The view is one of them, so an
  event touching nothing it reads costs no recompute *and* no bytes — the loop
  compares `sig-version` and skips the frame entirely.
- **Undo and redo**, below.

What it costs: butterfly's graph is a single-threaded singleton, so every signal
write happens on the loop's fiber. Background work wakes the UI with
`term-post` and lets the loop do the writing — it must not reach into the graph
from another fiber.

### Undo

`dispatch` normally returns `()`. Returning `:undo` or `:redo` asks the runtime
to rewind or replay the graph instead:

```irij
if (key? ev "u")
  :undo
else
  …
```

The history lives in the loop's own recursion, **not** in a signal — it has to,
because `restore` rewinds the whole graph, so a history kept inside it would be
rewound too and the second step back would have nowhere to go. It's bounded
(`history-limit`, 200), since a snapshot holds the whole graph.

`examples/reactive-todo.irj` is the todo list rebuilt this way, with working
undo/redo.

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
irij test        # 244 tests, no terminal required
```

## Examples

```
irij run examples/todo.irj            # needs a tty
irij run examples/reactive-todo.irj   # the same app on signals, with undo
```

A list you can move around, tick off and filter, with a progress bar and a help
line — about 150 lines, most of it the model. The reactive version splits that
model into signals and adds undo/redo.

## License

MIT
