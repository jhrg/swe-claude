<!-- Paste the body below into CLAUDE.md in place of the {{CONVENTIONS}} placeholder,
     and drop or demote this H2 so CLAUDE.md keeps a single `## Conventions` heading. -->

## Conventions — Arduino / PlatformIO (C/C++)

**Stack:** C++14 (verify each target toolchain actually builds it — older AVR
toolchains default to gnu++11 — and bump to C++17 only if every target board supports
it; check before assuming), Arduino framework via PlatformIO. Target board(s)/MCU family:
`{{BOARD(S) — e.g. ATmega328P (Uno), ESP32-WROOM, SAMD21}}`.

### Project layout & `platformio.ini`

- Pin exact versions: `platform = espressif32@6.x.x`, not a bare `platform =
  espressif32`. An unpinned platform/toolchain update is a silent way to break a build
  that worked yesterday.
- One `[env:...]` per physical target, plus a `[env:native]` (`platform = native`, no
  `board =`; uses the host compiler) for anything hardware-independent — this is what makes off-device unit
  testing possible at all.
- Build flags: `-Wall -Wextra` always on; move to `-Werror` once the codebase is
  warning-clean, and don't silence a warning with a pragma without a comment saying
  why.
- Keep algorithmic/business logic in plain C++ (`lib/` or `src/`) that does **not**
  include `Arduino.h`. Confine `Arduino.h`, `digitalRead/Write`, `Serial`, `Wire`,
  `SPI`, etc. to a thin hardware-adapter layer. Logic that doesn't touch a register
  directly has no business including the Arduino core.

### Language & style

- No exceptions, no RTTI — this matches the Arduino core's default
  `-fno-exceptions -fno-rtti` build. Report failure with status codes / enums, not
  `throw`.
- Fixed-width integer types (`uint8_t`, `int16_t`, `uint32_t`, ...) for anything tied
  to register widths, protocol fields, EEPROM/flash layout, or wire formats. Plain
  `int`/`long` are fine for loop counters and ordinary arithmetic, nowhere else.
- No dynamic allocation after `setup()` on RAM-constrained boards (classic AVR,
  SAMD21, anything in the 2–32 KB RAM range): no `new`, no `malloc`, no `String`
  concatenation in a loop. Heap fragmentation on a device with a few KB of RAM is a
  real, field-reported failure mode, not a style nitpick. Use fixed-size buffers
  (plain arrays, or `std::array` where the toolchain ships the C++ standard library —
  classic AVR's typically doesn't) instead. On boards with a real heap (ESP32, Teensy, etc.) this relaxes,
  but allocation still never happens inside an ISR or a tight timing loop.
- Avoid Arduino `String` in library/logic code — use `char[]` buffers or fixed spans.
  `String` is acceptable in top-level sketch glue code, not in reusable modules.
- Long string literals and lookup/constant tables go in flash (`PROGMEM`, `F()`) on
  AVR targets rather than RAM; skip this where the target has ample RAM and no flash
  pressure.
- Every variable shared between an ISR and the main loop is `volatile`. ISRs do the
  minimum possible work (set a flag, copy a value) and defer everything else to
  `loop()`.
- No blocking `delay()` in library code or anywhere with concurrent responsibilities —
  use `millis()`/`micros()`-driven state machines so multiple timed behaviors can
  coexist without starving each other.
- Pin numbers and board-specific constants are named `constexpr`s in one place (e.g.
  `pins.h`), never magic numbers scattered through `.cpp` files.

### Testing

- Hardware-independent logic gets a unit test under `test/`, using PlatformIO's
  Unity-based runner against `env:native` (`pio test -e native`) — tests run on the
  host, no board required.
- Anything that must touch real silicon to verify gets a fake/mock behind the
  hardware-adapter interface; on-target runs (`pio test -e <board>`) are for
  integration checks, not for exercising every logic branch.
- New logic-layer code isn't done without a native test. A bug fix isn't done without
  a regression test that fails before the fix and passes after.

### Static analysis & formatting

- `pio check` runs clean (cppcheck/clang-tidy backend) before a change is considered
  finished, or a new warning is justified inline with a comment.
- `clang-format` (config committed to the repo) is the formatting source of truth —
  don't hand-format around it.

### Documentation

- Every public function/class in `lib/` gets a one-line comment: what it does, units
  where relevant, and any hardware precondition (e.g. "call after `Wire.begin()`").
