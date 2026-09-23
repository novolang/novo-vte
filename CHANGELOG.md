# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number; see [Version numbers in the Orbit package
registry](https://novo-lang.org/docs/registry/semver.html).

## Unreleased

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
