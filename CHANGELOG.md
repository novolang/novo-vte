# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number; see [Version numbers in the Orbit package
registry](https://novo-lang.org/docs/registry/semver.html).

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
