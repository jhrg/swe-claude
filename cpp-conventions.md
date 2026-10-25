<!-- Paste the body below into CLAUDE.md in place of the {{CONVENTIONS}} placeholder,
     and drop or demote this H2 so CLAUDE.md keeps a single `## Conventions` heading. -->

## Conventions — C++ (CMake / CTest)

**Stack:** C++14 on Linux and macOS, built with CMake, tested with CTest. C++14 is the
baseline — verify each target compiler actually builds it, and bump to C++17 only if
every target compiler supports it; check before assuming. Compilers/platforms:
`{{COMPILERS & PLATFORMS — e.g. GCC 9+ and Clang 12+ on Linux; Apple Clang on macOS (arm64 + x86_64)}}`.
Unit-test framework: `{{FRAMEWORK — e.g. Catch2, GoogleTest, CppUnit}}`.

### Build & layout (CMake)

- Set `CMAKE_CXX_STANDARD 14`, `CMAKE_CXX_STANDARD_REQUIRED ON`, and
  `CMAKE_CXX_EXTENSIONS OFF`. Don't hardcode `-std=` in flags. Turning extensions off
  means code that only compiles under `gnu++14` fails for everyone, not just on
  someone else's machine.
- State `cmake_minimum_required(VERSION {{x.y}})` and don't use CMake features newer
  than the oldest CMake version CI builds with.
- Target-based CMake only: `target_include_directories`, `target_link_libraries`,
  `target_compile_options` with explicit `PUBLIC`/`PRIVATE`. No global
  `include_directories`, `add_definitions`, or `link_directories`.
- Out-of-source builds only; the build directory is never committed.
- Build the code as library target(s) that the executables **and** the tests link
  against — tests never recompile the sources. Layout: public headers in
  `include/<project>/`, implementation in `src/`, tests in `tests/`.
- Warnings: `-Wall -Wextra` always on. Move to `-Werror` once the codebase is
  warning-clean, as an opt-in CMake option that CI turns on. Don't silence a warning
  with a pragma without a comment saying why.
- Set `CMAKE_EXPORT_COMPILE_COMMANDS ON` so clang-tidy and editors see the real flags.

### Language & style

- RAII everywhere. No naked `new`/`delete`, and no `malloc`/`free` except to wrap a C
  API. Owning pointers are `std::unique_ptr` (`std::make_unique` exists in C++14);
  `std::shared_ptr` only when ownership is genuinely shared. Raw pointers and
  references are non-owning, always.
- `std::string` and standard containers over `char*` and C arrays. No `strcpy`,
  `sprintf`, or `gets`; use `snprintf` with its return value checked, or `std::string`.
- Exceptions are allowed for errors at API boundaries: derive from `std::exception`,
  throw by value, catch by `const&`. Destructors are `noexcept`. Never let an exception
  escape across a C ABI, a thread boundary, or a callback invoked from C code. Don't
  use exceptions for routine control flow (not-found, end-of-input). If this project
  builds with `-fno-exceptions`, replace this bullet with status codes / enums.
- `<cstdint>` fixed-width types for file formats, wire formats, and anything whose size
  is part of the contract; `std::size_t` for sizes and indices. No mixed
  signed/unsigned comparisons without an explicit, commented cast.
- Never read or write binary data by casting a struct or pointer. Serialize field by
  field with explicit endianness.
- `const` by default; `override` on every virtual override; a virtual destructor on
  every polymorphic base; `nullptr`, never `NULL` or `0`; `enum class` over plain
  `enum`; `constexpr` over `#define` for constants.
- Headers are self-contained (each compiles on its own), use include guards
  (`#pragma once` is fine), and never contain `using namespace`. Include what you use;
  forward-declare where that avoids an include.
- C++17 library/language features are off-limits at the C++14 baseline: no
  `std::optional`, `std::variant`, `std::string_view`, `std::filesystem`, structured
  bindings, or `if constexpr`. If one is truly needed, that's a decision to record as
  an ADR, not a quiet workaround.

### Portability (Linux & macOS)

- Standard C++ and POSIX only. Anything available on just one OS (`<endian.h>`,
  `<malloc.h>`, `strlcpy`/`strlcat`, `qsort_r`, `pthread_setname_np`, `memrchr`, the
  GNU-vs-XSI `strerror_r`) goes behind a small wrapper in one file, with
  `#if defined(__linux__)` / `#if defined(__APPLE__)` branches, each exercised by a test.
- Plain `char` signedness differs across platforms (e.g. x86_64 vs aarch64 Linux). Use
  `signed char`, `unsigned char`, or `uint8_t` wherever it matters.
- No GNU-only compiler extensions (statement expressions, VLAs, bare `__attribute__`).
- Never hardcode `.so` or `lib` prefixes (macOS uses `.dylib`). Use
  `$<TARGET_FILE:...>` and CMake's RPATH handling, not `LD_LIBRARY_PATH`;
  `DYLD_LIBRARY_PATH` is stripped by macOS SIP when a protected binary such as
  `/bin/sh` is in the process chain, so test scripts must not depend on it.
- macOS filesystems are usually case-insensitive and Linux's are case-sensitive:
  `#include` paths and file names must match case exactly.
- Parallelism: `cmake --build . -j` and `ctest -j`, not `nproc` (Linux-only; macOS is
  `sysctl -n hw.ncpu`).

### Testing

- Every test is registered with CTest (`enable_testing()` / `add_test`).
  `ctest --output-on-failure` is the single command that runs everything; a test CTest
  doesn't know about doesn't exist.
- Unit tests live in `tests/` and use `{{FRAMEWORK}}`, linked against the library target.
- **Baseline tests** — where the output is a text artifact (serialized response,
  generated metadata, CLI output) — compare actual output to a checked-in `*.baseline`
  file with `diff -u`, fail on any difference, and print the diff. Baselines change
  only through one deliberate update mode that rewrites them instead of comparing:
  `{{UPDATE MECHANISM — e.g. -DUPDATE_BASELINES=ON or UPDATE_BASELINES=1 ctest}}`.
  Never hand-edit a baseline. Update mode is off by default and never on in CI. Review
  `git diff` on baseline files like source code; a baseline change in a PR needs a reason.
- Test drivers are portable shell scripts in `tests/`: `#!/bin/sh`, `set -eu`, POSIX
  only. No bashisms, and no GNU-only flags (`sed -i`, `readlink -f`, `date -d`,
  `echo -e`, `grep -P`); use `printf`, and write to a temp file then `mv` instead of
  `sed -i`. Scratch space is `mktemp -d "${TMPDIR:-/tmp}/name.XXXXXX"`, removed by a
  `trap`.
- CMake passes the script everything it needs as arguments (`add_test(NAME x COMMAND
  ${CMAKE_CURRENT_SOURCE_DIR}/x.sh $<TARGET_FILE:tool> <baseline-dir>)`); scripts don't
  hardcode paths or guess the working directory. Prefer a script file over a
  quote-heavy `sh -c "..."` inline in `CMakeLists.txt` — nested quoting there is
  fragile and hard to review.
- Tests are deterministic and independent: no reliance on test order, wall-clock time,
  network, or a shared scratch location, so `ctest -j` is always safe. Fixed seeds only.
- New logic isn't done without a test. A bug fix isn't done without a regression test
  that fails before the fix and passes after.
- The suite also runs clean under ASan + UBSan on Linux and macOS (a CI job or a
  documented local build type). LeakSanitizer isn't available on macOS, so leak
  checking is Linux-only.

### Static analysis & formatting

- `clang-format` (config committed to the repo) is the formatting source of truth —
  don't hand-format around it.
- `clang-tidy` with a committed `.clang-tidy` runs clean before a change is considered
  finished, or a new finding is suppressed with a `NOLINT` and a comment saying why.

### Documentation

- Every public function/class in `include/` gets a one-line Doxygen-style comment: what
  it does, ownership/lifetime of pointer or reference arguments and returns, units where
  relevant, whether it can throw, and thread-safety when it isn't obvious.
