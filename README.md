# novo-vte

The shared xterm / VT escape-sequence parser and grid model.
Extracted from `orbit/novoterm` so `orbit/novomux` consumes the exact
same parser, which is what removes the "novomux inside novoterm
renders differently than novomux inside xterm" class of conformance
bug.

Pure Novo — `std.array` through `@ffi` and nothing else — so it runs
headless, with no PTY, no window and no GPU.  That is why its `@test`
suite (`src/vte_grid_tests.nv`, run by
`tests/orbit/test_novo_vte_grid.sh`) is the umbrella's coverage for
the shared model: novoterm's own tests link `render.nv`'s X11 + wgpu
surface and cannot run in a sweep.

Two modules:

| Module | What it holds |
|---|---|
| `grid` (`src/grid.nv`) | the cell store, the cursor, scroll regions, the alt screen, and the cell packing / unpacking helpers |
| `parser` (`src/parser.nv`) | the byte-at-a-time state machine: ground, ESC, CSI, OSC, DCS, UTF-8 continuation |

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
written, C-side, through the session palette.  Consumers therefore
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

- **`orbit/novoterm`** — GPU renderer.  Packs the attribute byte into
  the instance's `z` word; the WGSL fragment stage branches on bold,
  underline, crossed-out and the bar cursor, and applies reverse at
  the vertex stage by swapping fg and bg.  Italic is stored but not
  drawn: a skew is vertex-stage geometry rather than a fragment
  branch.
- **`orbit/novomux`** — text renderer.  `emit_sgr_attrs` re-emits one
  SGR per on-bit to the host terminal, so every rendition this table
  lists survives a mux pane, italic included.

## Tests

`tests/orbit/test_novo_vte_grid.sh` (`sec_novo_vte_grid`) runs
`src/vte_grid_tests.nv`; `tests/orbit/test_novo_vte_units.sh`
(`sec_novo_vte_units`) runs the unit suite.  Both are in the umbrella
sweep.
