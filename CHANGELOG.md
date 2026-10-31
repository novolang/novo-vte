# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number; see [Version numbers in the Orbit package
registry](https://novo-lang.org/docs/registry/semver.html).

## 0.4.1 — unreleased

### Added

- DECSET and DECRST 2026 (synchronized update) are tracked; `grid_sync_update` reports whether an update is open, so a renderer can hold a frame back until the end marker.
- DECRQM for a DEC private mode (`CSI ? Ps $ p`) is answered with DECRPM for ?7, ?25, ?1049, ?2004 and ?2026 (1 set, 2 reset) and with 0 for any other mode.
- `grid_resize_guard` and `grid_sync_since`: two stamps the renderer owns.  While the resize guard is up on the alternate screen a character at the right margin does not wrap.

### Fixed

- DECSTBM homes the cursor to (0, 0) when the margins are accepted, as the VT100 and xterm do.
- A count of 0 in CUU, CUD, CUF, CUB, CNL, CPL, HPR, VPR, ICH, DCH, IL, DL, ECH, SU and SD is read as 1, as xterm does.
- A resize that changes the column count keeps the cursor on the cell it stood on in the reflowed text, where it went to the bottom row.

## 0.4.0 — 2026-10-28

The package builds with no migration warning under toolchain 0.22.0 and
needs Novo 0.22.0 or later.  The requirement on `ansi-nv` is `^0.2.2`,
whose `var` markers at every call site the 0.21.1 compiler asks for (E2050)
are in place; the earlier 0.2.x versions drew those warnings into this
package's build.  The package no longer declares `@ffi`: the session
palette and the default colours are set through `grid` functions, which
changes what a host that pushed them through `window.nv` calls.  Below
`1.0.0` that is the reason for the minor number; the entry is the first
one under Unreleased, with the callers it touches.

The test suites cover every line under `src/`.

## 0.3.1 — 2026-09-26

The functions that build a list or a buffer in place declare the name
they write through `var`, and a value a caller keeps is copied where
it is handed on, as toolchain 0.11 requires of a list born under
`let`.  On 0.10.0 the markers change nothing.  No function's answer
changed.

## Unreleased

- **No `@ffi` left: the package is `host` with a `[mutate]` budget and
  publishes without `--allow-effect-budget`.**  The seven externs are
  gone.  The four `std.array` re-declarations in `grid` and `parser`
  are deleted and both modules call the stdlib's `array_i64_*` rows
  directly: since 0.9.0 they are `[]`, and a package build shows the
  standard library to its sibling modules, so the reason the copies
  were kept no longer holds.  The three terminal externs
  (`novo_term_palette_lookup`, `novo_term_default_fg_get`,
  `novo_term_default_bg_get`) read C globals; the same state is now
  module state in `grid`.  Sixty-four public functions that declared
  `[io, ffi]` for those externs now declare `[]`, and the ones that
  read the session colours (`pack_word0_idx`, `grid_sgr_reset`,
  `parser_feed_byte`) declare `[mutate]`.  Cell layout and every
  answer are unchanged: the palette seeds the same xterm table (the
  16 ANSI colours, the 6 x 6 x 6 cube, the 24-step gray ramp) and the
  defaults start at `0xAAAAAA` on `0x000000`.

- **Breaking for a host that sets the session colours: seven new
  `grid` functions replace the C runtime's setters.**
  `palette_lookup`, `palette_set`, `palette_reset`,
  `session_default_fg`, `session_default_bg`,
  `session_set_default_fg` and `session_set_default_bg` are the
  palette and the SGR 39 / 49 defaults.  A program that pushed a theme
  or handled OSC 4 / 10 / 11 / 104 through `window.nv`'s
  `novo_term_palette_set_h`, `_reset_h`, `novo_term_default_fg_set_h`
  or `_bg_set_h` no longer reaches the parser, which reads this
  package's state, and calls the `grid` functions instead.  A program
  that only feeds bytes and reads cells changes nothing.
  `orbit/novoterm` and `orbit/novomux` call them.

- **Eight public functions take their grid `var`, and a call marks it.**
  `grid_put_char`, `grid_backspace`, `grid_tab`, `grid_carriage_return`,
  `grid_line_feed`, `grid_erase_line` and `grid_erase_display` declare
  `var g: Grid`, and `parser_feed_byte` declares `var g: Grid` beside
  its `var p: Parser`: each writes through the grid it is handed, so the
  caller writes `grid.grid_put_char(var g, 65)` and
  `parser.parser_feed_byte(var p, var g, b)`.  An unmarked call is the
  warning E2050 on Novo 0.21.1 and an error in a later release.  The
  signatures changed and no function's answer did.  This is a breaking
  change for callers, so below 1.0 the release that carries it bumps
  the minor number, and it needs Novo 0.21.1 or later for the marker.
  `orbit/novoterm` and `orbit/novomux` write the markers.

- **The escape-sequence parser is `ansi-nv`'s.**  `src/parser.nv` no
  longer carries a state machine: it depends on `ansi-nv ^0.2.0`, feeds
  each byte to `vtparse.feed`, reads what the byte asked for with
  `vtparse.action_of`, and maps that action onto a grid operation.  The
  twenty-five functions that read bytes — the six `feed_*` state
  handlers, the UTF-8 decoder, the parameter collector and the slot
  accessors under them — are gone, and what is left is the glue a grid
  model owns: the CSI final-byte table, the DEC private modes, the SGR
  fold onto a cell's two words, the replies a terminal owes the host
  for DA, DSR and CPR, and the OSC payload it holds for its caller.  The public surface is unchanged, and so is
  every existing test.  Three answers change, each one a defect in the
  parser that is gone: `CSI 38:2::R:G:B m` sets the colour it names
  rather than reading the colon form as one long number; `CSI 58 ; 2 ;
  0 ; 0 ; 0 m` leaves the other attributes standing rather than reading
  its channels as SGR 0; and a C0 control inside a control sequence
  acts where ECMA-48 §5.4 says it does.

- **The parser is held as a value, not packed into an array.**
  `ansi-nv` 0.2.0's parser is a seventy-two-byte value over
  fixed-capacity buffers, and a `Parser` keeps it in a flat buffer of
  one, which is one owned allocation written and read in place.  What
  went with the move: the twenty-slot round trip that wrote the
  parser's state, parameters and intermediates into an `i64` array
  after every byte and read them back before the next one, the two
  functions that were its ends, the twelve-case pair that encoded the
  machine's state as an integer and decoded it again, and the
  192-entry parameter region the array carried for them.  `Parser` is
  two side-arrays and the machine, and `parser_state` reads the state
  off the machine rather than out of a slot.  Measured by
  `orbit/novoterm`'s `bench_parse` probe on the same 10 040-byte
  stream, best of four runs on one workstation: 10.4 MB/s before,
  17.1 MB/s after, against the suite's 50 KB/s bar.

- **A sequence that breaks a ceiling is dropped rather than
  truncated.**  A control sequence may carry 16 parameters, counting a
  colon group as one, and a parameter may hold 65534; both are the
  state machine's own capacity, and a sequence over either is dropped.
  Three colours in the colon form is eighteen entries and is dropped;
  the semicolon form of the same three is fifteen and still fits.  A
  sequence may carry two intermediate bytes.  An OSC payload past the
  8192-byte cap is refused at the byte that passes it, the rest is
  consumed to the terminator, and nothing is handed to the caller — so
  a window title or a clipboard write that is too long has no effect
  rather than a truncated one.

- **The replies a device query gets are `vtquery`'s bytes.**  DA,
  secondary DA, DSR and CPR were four escape sequences spelled out as
  string literals beside a parser that reads the same four.  They are
  now `vtquery.write_reply`, which produces them byte for byte —
  `ESC [ ? 62 ; 22 c`, `ESC [ > 0 ; 1 ; 0 c`, `ESC [ 0 n` and
  `ESC [ <row> ; <col> R` are unchanged on the wire.

- **The contract suite moved out of `src/`** — `vte_grid_tests.nv` (758
  lines) now lives under `tests/`, which is where
  `orbit/package-template` puts a suite and for the reason it gives:
  `src/` ships whole, so a consumer compiled it, the published archive
  carried it, and `novo doc` printed a third module section for it — a
  reader opening the package's page to learn what it does was shown its
  test suite beside `grid` and `parser`.  The reference is now two
  modules.  Nothing a consumer calls changes.

## 0.2.3

- **Declares its layer**: `layer = "host"` in the manifest — the grid declares `[io, ffi]` today for memory-only externs, which two compiler-side filings own; when they close the package becomes `core` with no change to its layout, and `novo pkg publish` now checks the code against that budget.  The layers are described under Design in the [publishing guide](https://novo-lang.org/docs/publishing.html#design).
- `bits.*` calls are the operators they lower to (`&`, `|`, `^`, `<<`, `>>>`, `~`), rewritten by `novo rewrite --bits-to-operators` where the checker proves the operands `Int`; every test vector byte-identical.  Sources reformatted to the canonical form.
- Relicensed from MIT to Apache-2.0, so every first-party package carries
  one licence and the patent grant that comes with it.  The copyright
  holder is unchanged and no third-party code is involved; releases
  already published as 0.2.2 and earlier remain under MIT.

## 0.2.2

The public surface is named. Adding this package no longer produces a
wall of privacy warnings, and its page on the registry has entries.
Nothing that worked stops working — this is a compatible addition, so a
patch bump.

- **Ninety-four items are `pub`**, and they are what novoterm and
  novomux call plus what this README already promised: the `Grid` and
  `Parser` types, the cell readers and packers, the SGR state, the
  control characters, the ECMA-48 editing functions in full pairs, the
  mode accessors, the scrollback view, and the parser's two output
  channels. Before this, a consumer got one
  `NOVO_PACKAGE_PRIVACY` warning per call — 589 building novoterm, 152
  building novomux. Both are now zero.
- **Everything else stays private on purpose**, and the README says
  which classes and why: the `@ffi` re-declarations, the cell packers
  below `pack_word0_idx`, the state-machine slot accessors, the
  scrollback ring's construction, and the mode setters the parser
  drives from the stream.
- **Every public item carries its documentation**, and five of them
  carry a compiled example — the parser's feed-and-drain loops for OSC
  and for device replies, and the grid's read-back.
- **Every public function is named by a test.** The suite went from 9
  assertions to 30, covering the geometry, the cells, the SGR state,
  the editing family, the modes, the scrollback view and both parser
  channels.

## 0.2.1

Documentation: the reference is generated from the code, and the
examples in it are doctests.  No code changed — every cell, every
escape sequence and every FFI symbol is what 0.2.0 shipped.

- **Both modules carry a worked example, and both run.**  A grid built,
  written into and dumped; a parser fed `ESC [ 2 ; 1 H` one byte at a
  time, moving the cursor a `B` then lands under.  A fenced `novo`
  block in a documentation comment is compiled by `novo doc` and run by
  `novo test src/grid.nv` and `novo test src/parser.nv`, so an example
  that stopped being true is a failing test rather than a reader's
  afternoon.
- **The README says where the reference stands.**
  [The package's page](https://novo-lang.org/packages/novo-vte) carries
  the two modules' documentation and no entries, because nothing here
  is marked `pub` yet — every consumer already calls these functions
  and is warned on each call.  Naming the public surface is a decision
  about what this package promises across roughly two hundred
  functions, and it has not been taken; the README no longer leaves a
  reader to discover that from a build log.

## 0.2.0

**Built and tested against the 0.7.1 toolchain, and it is the floor.**
That is the break, and it is why this is a minor bump rather than a
patch: 0.6.0 made `[ffi]` required on an `@ffi` declaration, so a
consumer that changes nothing and stays on an older toolchain stops
building.  `0.1.0`, for its part, does not compile on 0.7.1 at all.

- **Builds again.**  A function annotated `@ffi` must declare `[ffi]` in
  its effect row — the effect is what marks a call into foreign code the
  checker cannot see through.  `0.1.0`'s seven declarations in `grid.nv`
  declared `[io]` alone, so every consumer failed with `E3003` before
  reaching its own code.
- **Italic and strikethrough are stored and re-emitted.**  The attribute
  mask carries `attr_italic()` (bit 4) and `attr_strike()` (bit 5), and
  each bit is an independent switch: `CSI 1;4m` is bold *and*
  underlined, and `CSI 24m` clears underline alone.
- **OSC payloads up to 8192 bytes.**  512 truncated a real OSC 52
  clipboard write, whose base64 body carries three quarters of its
  length in text.
- **The source is canonical.**  `novo fmt` prints these files unchanged.

## 0.1.0

First release: the byte-at-a-time VT state machine, the two-word cell
format, the grid with scroll regions and an alt screen.
