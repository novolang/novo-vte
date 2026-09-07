# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number; see [Version numbers in the Orbit package
registry](https://novo-lang.org/docs/registry/semver.html).

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
