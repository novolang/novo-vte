# Code improvement candidates — novo-vte

_Structural / readability review (auto-generated). Each item cites `file:line`._

The parser/grid split is generally well-factored (small, named helpers; the
`slot_*()` / `attr_*()` accessor-fn "constants" are an intentional design to
keep parser/grid state in an FFI-backed i64 array and avoid a struct clone per
byte — see the leak note at `grid.nv:145`). The items below are the few
structural offenders.

## 1. Long functions

- **`fn dispatch_csi` — `src/parser.nv:293` (148 lines). Worst offender.**
  A long `if / elif` chain branching on `final_byte`. The single-line cursor
  arms (CUU/CUD/CUF/CUB/CUP/…, `src/parser.nv:304`–`370`) are fine, but the
  chain buries four multi-line handlers that should be their own fns:
  - `'c'` DA query (`src/parser.nv:377`–`389`) → `fn handle_da(p, g)`.
  - `'n'` DSR (`src/parser.nv:393`–`403`) → `fn handle_dsr(p, g)`.
  - `'r'` DECSTBM (`src/parser.nv:405`–`411`) → `fn handle_decstbm(p, g)`.
  - `'q'` DECSCUSR (`src/parser.nv:420`–`429`) → `fn handle_decscusr(p, g)`.
  Reads as a protocol-opcode table; the CSI final bytes themselves are
  legitimate wire numerics (do not turn those into an enum).

- **`fn feed_esc` — `src/parser.nv:734` (90 lines).** if/elif on the byte after
  ESC. The IND/NEL/RI arms (`src/parser.nv:789`–`817`) are ~8 lines each with
  scroll-region logic → extract `handle_ind` / `handle_nel` / `handle_ri`.

- **`fn grid_resize` — `src/grid.nv` (~105 lines)** and `fn grid_enter_alt_screen`
  (~65). Moderate; the reflow walk in `grid_resize` is the candidate for
  splitting out a `reflow_logical_lines` helper (a `reflow_cells` helper already
  exists at `src/grid.nv`).

## 2. Long / heavy match arms

Same subjects as above — these live in `if/elif` chains rather than a `match`,
but the effect is identical: DA / DSR / DECSTBM / DECSCUSR arms inside
`dispatch_csi`, and IND/NEL/RI inside `feed_esc`, are the arms heavy enough to
delegate to named `handle_*` fns.

## 3. Integer codes that should be enums / consts

- **Parser state — `src/parser.nv:53`–`70`.** `st_ground()=0 … st_dcs_string()=5`
  are Int-returning accessor fns, and `parser_feed_byte` (`src/parser.nv:623`)
  dispatches with `st == st_ground()` / `elif st == st_esc()` … This is a finite
  set of state meanings; the header comment (`src/parser.nv:22`) already calls
  them "ParserState enum codes". Candidate for a real `enum ParserState`.
  _Caveat:_ the state is physically stored in the i64 slot array (slot 0) for the
  no-clone hot path, so an enum would need an int round-trip at the storage
  boundary — the current named-fn constants are a reasonable middle ground.
  Flag as low-priority / stylistic.
- DEC private-mode numbers (47/1047/1049/7/2004/1000/1002/1003/1006/25) in
  `dispatch_private_mode` (`src/parser.nv:441`) and the SGR codes in
  `apply_sgr_one` (`src/parser.nv:589`) are **wire numerics** — well-commented,
  do not convert.

## 4. Traits / generics opportunities

- **Near-duplicate string-consumer state fns.** `feed_osc_string`
  (`src/parser.nv:853`) and `feed_dcs_string` (`src/parser.nv:886`) implement the
  same BEL / `ESC \` terminator state machine — `feed_dcs_string` is
  `feed_osc_string` minus the payload accumulate. Factor the shared
  terminator-detection into one helper and let OSC pass an "accumulate" callback
  / flag. (No trait needed; plain shared helper.)
- The mouse-mode arms in `dispatch_private_mode` (`src/parser.nv:474`–`493`)
  repeat `if set then grid_mouse_mode_bit_on(g, N) else …_off(g, N)` four times
  → collapse to `grid_set_mouse_bit(g, bit, set)`.

## 5. Reinvented stdlib data structures

- The response queue (`src/parser.nv:214`–`227`) and OSC accumulator
  (`src/parser.nv:164`–`209`) are hand-rolled bounded byte buffers over an i64
  array — conceptually an `sb`/`ring`-style append-with-cap buffer. Reusing a
  stdlib byte buffer is blocked by the deliberate FFI-array + fixed-slot layout,
  so this is **noted, not recommended**.

## Summary
- Long functions: 3 (worst: `fn dispatch_csi` @ 148 lines)
- Long match arms: ~6 (DA/DSR/DECSTBM/DECSCUSR + IND/NEL/RI)
- Int-code APIs: 1 real (parser `st_*` states; storage caveat) — pub: 0
- Trait/generic opportunities: 2 (OSC/DCS consumer dedup; mouse-bit helper)
- Reinvented structures: 0 recommended (2 noted, blocked by FFI-array design)
- Overall priority: Split the 4 multi-line arms out of `dispatch_csi` and the 3 out of `feed_esc`; dedup `feed_osc_string`/`feed_dcs_string`. The `st_*` → enum move is optional given the intentional i64-slot storage.
