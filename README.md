# novo-vte

The shared xterm / VT escape-sequence parser and grid model.
It was extracted from novoterm so that novomux consumes the exact
same parser, which removes the "novomux inside novoterm renders
differently than novomux inside xterm" class of conformance bug.

Pure Novo, with no C binding of its own, so it runs headless: no PTY,
no window and no GPU.  The package is layer `host`; its only effect
beyond memory is `[mutate]`, for the session colours described below.
Feed it bytes, read cells back: everything about drawing them belongs to the consumer.

## Using it

It is a library: depend on it and `use` the module you need.

```
novo pkg add novo-vte
```

```novo
use grid
use parser
```

A grid is a rectangle of cells with a cursor and the current colours
riding on it; the parser turns a byte at a time into changes to one.
Both hold their cells in `std.array` buffers, so a caller drops what it
made, and `parser_feed_byte` declares `[mutate]` because an SGR reads
the session colours:

```novo
use grid
use parser

fn main() [io, mutate]
    var g = grid.grid_new(2, 3)
    var p = parser.parser_new()
    for b in [65, 27, 91, 50, 59, 49, 72, 66]   // A ESC [ 2 ; 1 H B
        g = parser.parser_feed_byte(var p, var g, b)
    print(grid.grid_to_string(g))               // @ 1,1 / A / B
    parser.parser_drop(p)
    grid.grid_drop(g)
```

```
novo pkg build                       # build the package
novo test tests/vte_grid_tests.nv    # 30 tests
```

## The reference

Every public function, its signature, its effect row and a worked
example of each entry point are on
[the package's page](https://novo-lang.org/packages/novo-vte),
generated from these sources at every publish. A list of names here
would be a second original, and the second original is the one that
goes stale.

## What is public

A hundred and one items across the two modules, and they are what this
package promises. The groups below are what the reference page is
organised by; the page itself carries the signatures.

| Group | Module | What it is for |
|---|---|---|
| `Grid`, `grid_new`, `grid_drop`, `grid_resize`, `grid_rows`, `grid_cols` | `grid` | make a screen, resize it, release it |
| `grid_get`, `grid_get_w1`, `grid_set`, `grid_set_w1`, `grid_set_pair` | `grid` | the two words of a cell, read and written |
| `cell_glyph`, `cell_attrs`, `cell_fg`, `cell_bg`, `pack_word0_idx`, `pack_word1`, `default_fg_rgb`, `default_bg_rgb` | `grid` | unpack a word into its fields, and pack one back |
| `attr_bold`, `attr_underline`, `attr_reverse`, `attr_italic`, `attr_strike` | `grid` | the five rendition bits, one per ECMA-48 switch |
| `grid_cursor_row`, `grid_cursor_col`, `grid_cursor_set`, and the `grid_*cursor_visible` / `grid_*cursor_shape*` accessors | `grid` | where the cursor is, whether it is drawn, and what shape |
| `grid_set_fg`, `grid_set_bg`, `grid_set_attrs`, `grid_add_attr`, `grid_clear_attr`, `grid_sgr_reset` | `grid` | the SGR state later writes inherit |
| `palette_lookup`, `palette_set`, `palette_reset`, `session_default_fg`, `session_default_bg`, `session_set_default_fg`, `session_set_default_bg` | `grid` | the session colours: the 256-entry palette an indexed SGR resolves through and the two defaults SGR 39 / 49 return to, which a host sets once and an OSC 4 / 10 / 11 / 104 changes |
| `grid_put_char`, `grid_backspace`, `grid_tab`, `grid_carriage_return`, `grid_line_feed` | `grid` | write a glyph, and the four control characters |
| `grid_erase_line`, `grid_erase_display`, `grid_erase_chars`, `grid_insert_lines`, `grid_delete_lines`, `grid_insert_chars`, `grid_delete_chars` | `grid` | the ECMA-48 editing functions, in full pairs |
| `grid_set_scroll_region`, `grid_scroll_up`, `grid_decsc`, `grid_decrc`, `grid_enter_alt_screen`, `grid_exit_alt_screen`, `grid_using_alt` | `grid` | scroll regions, the saved cursor, the alt screen |
| `grid_bracketed_paste`, `grid_mouse_mode`, `grid_mouse_sgr_code` | `grid` | the mode flags a renderer reads each frame, and the one function that decides whether a mouse event is forwarded |
| `grid_view_get`, `grid_view_get_w1`, `grid_view_offset`, `grid_scroll_view`, `grid_reset_view`, and the `scrollback_*` accessors | `grid` | reading the screen through the scrollback view, so scrolling costs the renderer nothing |
| `grid_to_string` | `grid` | the readable dump, cursor first, one line per row |
| `Parser`, `parser_new`, `parser_drop`, `parser_feed_byte`, `parser_state`, `st_ground` … `st_dcs_string` | `parser` | make a parser, feed it a byte, ask what state it is in |
| `parser_osc_pending`, `parser_osc_code`, `parser_osc_payload_len`, `parser_osc_payload_byte`, `parser_osc_clear` | `parser` | an OSC is handed to the caller to act on — this module never sets a window title itself |
| `parser_response_count`, `parser_response_byte_at`, `parser_response_clear` | `parser` | the replies the terminal owes the host, drained by the caller's pump |

**What stays private, so you know you are not missing it:**

- The cell packers below `pack_word0_idx` / `pack_word1`
  (`pack_word0`, `blank_word0`, `blank_word1`, `blank_cell`) and the
  index arithmetic around them.
- The parser's state-machine slot accessors (`slot_*`, `max_params`,
  `response_cap`, `osc_cap`) and its `feed_*` / `dispatch_*` arms —
  they are reached by feeding a byte, not by calling them.
- The scrollback ring's own construction (`scrollback_new`,
  `scrollback_push_row`, `scrollback_cap`, `scrollback_head`) — the
  ring is built by `grid_new` and read through the view.
- The mode setters the parser drives from the stream (autowrap, wrap
  pending, mouse-mode bits, per-row wrap flags): the escape sequence is
  the way in, and the accessor above is the way out.

## Limits

| Limit | Value | What it bounds |
|---|---|---|
| OSC payload | 8192 bytes | one `\e]…` payload after the `N;` prefix.  Sized for OSC 52, whose base64 body carries 3/4 its length in text — a ~6 KB selection.  A payload that passes the cap is dropped whole: the rest of it is consumed to the terminator and nothing is handed to the caller. |
| Reply channel | 64 bytes | the parser's answers to device queries (DA, DA2, DSR, CPR), drained by the caller's pump after each byte |
| Parameters | 16 per sequence | the numbers of one control sequence, counting a colon group as one.  A sequence with more is dropped. |
| Parameter value | 65534 | the largest number one parameter may hold.  A larger one is dropped rather than wrapped, because a wrapped value is a cursor move nobody asked for. |
| Intermediate bytes | 2 per sequence | the bytes between the parameters and the final byte |

The first two are fixed-size `i64` arrays inside `Parser`, so the OSC
cap costs 64 KB per parser — one per pane in a mux.  The last three are
the state machine's own capacity and cost nothing per parser: a
`Parser` holds the machine as seventy-two bytes.

Two modules:

| Module | What it holds |
|---|---|
| `grid` (`src/grid.nv`) | the cell store, the cursor, scroll regions, the alt screen, and the cell packing / unpacking helpers |
| `parser` (`src/parser.nv`) | what a sequence does to the grid: the CSI final-byte table, the DEC private modes, the SGR fold, the reply channel and the OSC payload |

The state machine under `parser` is
[ansi-nv](https://novo-lang.org/packages/ansi-nv)'s: a byte goes to
`vtparse.feed`, which answers the parser after it, `vtparse.action_of`
says what the byte asked for, and this package decides what the screen
does about it.  A terminal and a multiplexer
that disagreed about what a sequence *meant* is the bug this package
exists to prevent; one parser for what a sequence *says* is the same
argument one level down.

## The cell format

A cell is **two** `i64` words, not one — a single word cannot hold two
24-bit colours beside the glyph and the attributes.

| Word | Bits | Field |
|---|---|---|
| 0 | 0-15 | glyph (codepoint) |
| 0 | 16-23 | attribute mask |
| 0 | 24-47 | foreground, 24-bit RGB |
| 1 | 0-23 | background, 24-bit RGB |

Read with `cell_glyph(w0)`, `cell_attrs(w0)`, `cell_fg(w0)` and
`cell_bg(w1)`; write with `grid_set` (word 0), `grid_set_w1` (word 1)
or `grid_set_pair` (both at once).

`pack_word0_idx(glyph, fg_idx, attrs)` builds word 0 from a palette
INDEX rather than an RGB value, resolving the index through the
session palette.  It is named for the word it builds and takes only
the fields that word holds: a background needs `pack_word1` and a
second write, because a function returning one `Int` has nowhere to
put it.

**Colours arrive resolved.**  An indexed SGR colour (`CSI 38;5;Nm`,
`CSI 31m`) is turned into 24-bit RGB at the moment the cell is
written, through the session palette.  Consumers therefore
never look a colour up — a renderer reads RGB straight out of the
cell, and a live palette change affects only cells written after it.

## Character attributes

The attribute mask is 8 bits, and each bit is an INDEPENDENT switch,
per ECMA-48 §8.3.117.  `CSI 1;4m` is bold *and* underlined; `CSI 24m`
clears underline and leaves everything else standing.

| Bit | Value | Constant | SGR set / clear |
|---|---|---|---|
| 0 | 1 | `attr_bold()` | 1 / 22 |
| 1 | 2 | `attr_underline()` | 4 / 24 |
| 2 | 4 | `attr_reverse()` | 7 / 27 |
| 3 | 8 | — | reserved: novoterm's bar-cursor shape, OR'd in by the renderer at the cursor cell |
| 4 | 16 | `attr_italic()` | 3 / 23 |
| 5 | 32 | `attr_strike()` | 9 / 29 |

Set one bit with `grid_add_attr(g, bit)`, clear one with
`grid_clear_attr(g, bit)`.  `grid_set_attrs(g, mask)` replaces the
whole mask and is for a reset or a test that pins an exact value — an
SGR that names one rendition must not use it, because assigning the
mask is what made `CSI 1;4m` render underline without bold.

Bit 3 is not free.  A new rendition takes bit 6 or 7.

## Consumers

- **novoterm** — GPU renderer.  Packs the attribute byte into
  the instance's `z` word; the WGSL fragment stage branches on bold,
  underline, crossed-out and the bar cursor, and applies reverse at
  the vertex stage by swapping fg and bg.  Italic is stored but not
  drawn: a skew is vertex-stage geometry rather than a fragment
  branch.
- **novomux** — text renderer.  `emit_sgr_attrs` re-emits one
  SGR per on-bit to the host terminal, so every rendition this table
  lists survives a mux pane, italic included.

## Tests

```
novo test tests/vte_grid_tests.nv    # 30 tests
novo test src/grid.nv                # the documented examples in grid
novo test src/parser.nv              # the documented examples in parser
```

The library has no `ffi` row and no extern of its own.  The parser suite
re-declares one runtime extern with `@ffi`, `novo_term_palette_lookup`, to
check the 256-colour palette against the runtime's table, and that
declaration is private to `tests/parser_tests.nv`.

The suite drives the parser a byte at a time and reads the grid back,
so a failure names the escape sequence that produced the wrong cell.
Every public function is named by one of these tests, and the examples
in the documentation are tests too — an example that stopped being true
is a red run rather than a reader's afternoon. novoterm's conformance
fixtures drive the same two modules against recorded xterm output and
are the wider net.
