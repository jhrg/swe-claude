<!-- Paste the body below into CLAUDE.md in place of the {{CONVENTIONS}} placeholder,
     and drop or demote this H2 so CLAUDE.md keeps a single `## Conventions` heading. -->

## Conventions — Python 3 (pyproject / pytest)

**Stack:** Python `{{MIN PYTHON — e.g. 3.11}}`+ on Linux and macOS, packaged with
`pyproject.toml`, tested with pytest. The minimum version is the baseline for both the
language and the standard library; pick the oldest CPython still receiving security
fixes that every deployment target actually has, and check before assuming. Features
newer than the baseline are listed under **Version-gated features** below together with
the baseline way to do the same thing. Interpreters/platforms:
`{{INTERPRETERS & PLATFORMS — e.g. CPython 3.11–3.13 on Linux x86_64/aarch64 and macOS arm64}}`.
Environment/dependency tool: `{{TOOL — e.g. uv, or pip + venv with pip-tools}}`.

### Packaging & layout

- `pyproject.toml` is the single source of project metadata and tool configuration
  (ruff, mypy, pytest, coverage). No `setup.py`, `setup.cfg`, `requirements*.txt`
  edited by hand, or per-tool config files when `pyproject.toml` can hold the same
  settings.
- The minimum version is set in **one place**, `requires-python = ">={{MIN PYTHON}}"`,
  and every tool reads its target from there or is set to match it: ruff
  `target-version`, mypy `python_version`. That way a linter or type checker flags
  syntax and stdlib APIs newer than the baseline.
- `src` layout: the package lives in `src/{{PACKAGE}}/`, tests in `tests/`. Tests run
  against the installed package (`pip install -e .` / `uv sync`), never via
  `sys.path` manipulation or by running from the repo root.
- Every environment is a virtual environment. Never `pip install` into the system or
  Homebrew Python, and never `sudo pip`.
- Direct dependencies in `pyproject.toml` have lower bounds (and upper bounds only for
  a known break). Exact, hashed versions for every transitive dependency live in a
  committed lock file (`{{LOCK FILE — e.g. uv.lock, or requirements.lock from pip-compile}}`)
  that CI installs from.
- A runnable program is a `[project.scripts]` entry point or a `__main__.py`
  (`python -m {{PACKAGE}}`), not a loose script that imports from `src/` by path.
- No generated or environment files in git: `.venv/`, `__pycache__/`, `*.egg-info/`,
  `dist/`, `build/`, `.pytest_cache/`, `.mypy_cache/`.

### Language & style

- Type hints on every public function, method, and class attribute, and on anything
  non-trivial internally. `{{TYPE CHECKER — e.g. mypy --strict, or pyright strict}}`
  runs clean. `Any` and `# type: ignore[code]` (always with the specific code) only
  with a comment saying why.
- `@dataclass(frozen=True, slots=True)` (or `typing.NamedTuple`) for plain value
  carriers instead of dicts or tuples passed around by position. `Enum` over
  string or int constants.
- Never use a mutable default argument (`def f(x=[])`); use `None` and create it
  inside.
- Errors:
  - Raise specific exceptions, ideally a small project hierarchy rooted at one base
    class, so callers can catch the package's errors in one place.
  - No bare `except:`, and no `except Exception:` except at a top-level boundary (CLI
    `main`, a request handler, a worker loop), and there, log it with the traceback
    (`log.exception(...)`).
  - When translating an exception, chain it: `raise ConfigError(...) from e`.
  - Never swallow an exception (`except ...: pass`) without a comment saying why it is
    safe.
  - Don't use `assert` to validate input or enforce invariants that matter in
    production — `python -O` strips it. Raise instead.
- Every file, socket, lock, subprocess, or temp directory the code opens is managed by
  a `with` block (or `contextlib.ExitStack`). Don't rely on garbage collection to
  close things.
- `pathlib.Path` for filesystem paths, not string concatenation or `os.path` in new
  code.
- `datetime` objects are timezone-aware (`datetime.now(timezone.utc)`), never naive;
  `datetime.utcnow()` is deprecated. Pass a clock (a callable returning the time) into
  code that reads the time so tests can fix it. `decimal.Decimal` (constructed from a
  `str`, not a `float`) for money or any value that must be exact.
- Imports: absolute imports within the package, one module per line, no
  `from x import *`, no imports with side effects at module level (no work at import
  time beyond defining things). Module-level mutable state only with a comment saying
  who mutates it.
- Logging through `logging.getLogger(__name__)`. Library code never calls `print`, and
  never configures logging (no `basicConfig`) — only the entry point does. Use lazy
  `%`-style arguments in log calls (`log.debug("x=%s", x)`), not f-strings.
- Concurrency: pick one model per component — `asyncio`, threads
  (`concurrent.futures.ThreadPoolExecutor`), or processes
  (`ProcessPoolExecutor`) — and don't call blocking I/O from inside a coroutine (use
  `asyncio.to_thread`). Shared mutable state between threads is guarded by a lock that
  the code names in a comment. Threads don't speed up CPU-bound pure-Python code (the
  GIL); use processes for that.
- Off-limits: `eval`/`exec` on anything not written by this repo; `pickle`/`shelve`
  on untrusted data; `subprocess` with `shell=True` (pass an argument list);
  `yaml.load` without `SafeLoader`; monkey-patching third-party or stdlib modules
  outside tests.
- Syntax and stdlib APIs newer than the baseline are off-limits (ruff and the type
  checker, targeted at the baseline, flag most of them). Don't route around the
  baseline with `sys.version_info` branches or `try: import ...` fallbacks except in
  one compat module with a test for each branch. If a newer feature is truly needed,
  raising the baseline is a decision to record as an ADR.

### Version-gated features

Raising the baseline is one change: bump `requires-python`, ruff `target-version`, mypy
`python_version`, the CI interpreter matrix, and the lock file, then record an ADR.
Nothing in a row is allowed until the baseline is at or above that row's version. After
the baseline moves, prefer the newer form in new code; existing code is converted only
when touched for another reason, or as a `TASK-###` (`ruff check --select UP --fix`
does much of it mechanically).

| Need | Older way | Newer way (min. version) |
|---|---|---|
| Union / optional types | `Optional[X]`, `Union[X, Y]` | `X \| None`, `X \| Y` (3.10) |
| Branch on the shape of data | `if isinstance(...)` / `elif` chains | `match` / `case` with class and mapping patterns (3.10) — not as a substitute for a plain `if`/`elif` on values |
| Zip iterables that must be equal length | `zip`, then check lengths | `zip(a, b, strict=True)` (3.10) |
| Read TOML | Third-party `tomli` | `tomllib` (3.11) |
| Method returning its own class | `TypeVar` bound to the class | `typing.Self` (3.11) |
| Run and await a group of tasks | `asyncio.gather` | `asyncio.TaskGroup` (3.11), which cancels siblings on failure |
| Report several errors at once | A list attribute on a custom exception | `ExceptionGroup` and `except*` (3.11) |
| Add context to an exception in flight | Wrap and re-raise | `e.add_note(...)` (3.11) |
| Generic classes/functions and aliases | `TypeVar` / `Generic[T]` / `TypeAlias` | `class Box[T]:`, `def f[T](x: T)`, `type Alias = ...` (3.12) |
| Mark an override | Comment | `@typing.override` (3.12) |
| Chunk an iterable | Hand-written loop or `zip(*[iter(x)] * n)` | `itertools.batched(x, n)` (3.12) |
| Mark API deprecated | `warnings.warn(..., DeprecationWarning)` | `@warnings.deprecated` (3.13) |

Still off-limits at any baseline: free-threaded (no-GIL) or JIT builds as an assumption,
and any interpreter other than CPython (e.g. PyPy), unless the project lists it above.

### Portability (Linux & macOS)

- Always name the encoding: `open(path, encoding="utf-8")`, `Path.read_text(encoding="utf-8")`,
  `subprocess.run(..., text=True, encoding="utf-8")`. Before Python 3.15 the default
  comes from the locale, so it differs between machines (and between a shell and a
  cron job or container). Ruff's `PLW1514` catches the omission.
- Don't depend on the default locale for anything machine-readable (number formatting,
  `str.lower` is fine but `locale.format_string` is not); read times as UTC.
- The interpreter is `python3`, never `python` (which may be missing or Python 2 on
  some systems). Scripts start with `#!/usr/bin/env python3`, and the tooling runs
  inside the venv (`uv run ...`, or `.venv/bin/python -m ...`).
- `multiprocessing` default start method differs: `spawn` on macOS, `fork` on Linux
  before 3.14 (`forkserver` from 3.14). Code that uses processes must work under
  `spawn`: everything sent to a worker is picklable, worker functions are importable
  module-level functions, and the entry point is guarded by
  `if __name__ == "__main__":`. Set the start method explicitly if behavior depends on
  it.
- macOS filesystems are usually case-insensitive and Linux's are case-sensitive: module
  names, resource names, and file names must match case exactly. An import that works
  on a Mac can fail on Linux.
- Package data is read with `importlib.resources.files(...)`, never by building a path
  from `__file__`.
- Subprocesses get an argument list, not a shell string, and don't assume GNU tools
  (`sed -i`, `readlink -f`, `date -d`) are on the `PATH`. Temp space comes from
  `tempfile` (respects `$TMPDIR`), never a hardcoded `/tmp/...`.
- Write `\n` explicitly in machine-readable output, and open text files for writing
  with `newline="\n"` when the exact bytes matter.

### Testing

- `{{TEST COMMAND — e.g. uv run pytest, or python -m pytest}}` is the single command
  that runs every test. A test pytest doesn't collect doesn't exist; configure
  `testpaths` and `--strict-markers` in `pyproject.toml`.
- Unit tests live in `tests/`, use plain `assert` with pytest's introspection (not
  `unittest.TestCase`-style `assertEqual` in new code), fixtures over setup/teardown
  methods, and `@pytest.mark.parametrize` over loops in a test. Mocking
  (`unittest.mock`, `monkeypatch`) is for collaborators the test can't run for real —
  not for the code under test.
- **Baseline tests** — where the output is a text artifact (serialized response,
  generated metadata, CLI output) — compare actual output to a checked-in `*.baseline`
  file under `tests/`, fail on any difference, and print a unified diff
  (`difflib.unified_diff`). Baselines are read and written as UTF-8 with `\n` line
  endings. They change only through one deliberate update mode that rewrites them
  instead of comparing: `{{UPDATE MECHANISM — e.g. a --update-baselines pytest option
  defined in conftest.py}}`. Never hand-edit a baseline. Update mode is off by default
  and never on in CI. Review `git diff` on baseline files like source code; a baseline
  change in a PR needs a reason.
- Tests that need files use pytest's `tmp_path`, never a fixed path or the working
  directory. Tests that start servers bind to port `0` and read back the port.
- Tests are deterministic and independent: no reliance on test order, wall-clock time
  (inject a fixed clock), network, the default locale/encoding/time zone, environment
  variables the test didn't set (`monkeypatch.setenv`), or module-level state left over
  from another test — so `pytest -n auto` (pytest-xdist) and random ordering are always
  safe. Fixed seeds only.
- New logic isn't done without a test. A bug fix isn't done without a regression test
  that fails before the fix and passes after.
- Coverage is measured with `coverage.py` / `pytest-cov` in CI, branch coverage on:
  `{{COVERAGE GATE — e.g. report only, or fail_under = 85}}`. A gate is a floor, not a
  goal; don't write tests that only execute lines.
- CI runs the suite on every interpreter in the matrix, including the baseline version
  — passing on the newest Python says nothing about the oldest.
- Warnings are errors in tests (`filterwarnings = ["error"]`), with specific
  third-party warnings ignored by name and a comment saying why. A
  `DeprecationWarning` today is a break on the next Python release.

### Static analysis & formatting

- `ruff format` is the formatting source of truth — don't hand-format around it.
- `ruff check` (rule selection committed in `pyproject.toml`, including at least `E`,
  `F`, `I`, `B`, `UP`, `PLW1514`) and `{{TYPE CHECKER}}` run clean before a change is
  considered finished, or a new finding is suppressed with `# noqa: CODE` /
  `# type: ignore[code]` and a comment saying why. Never a bare `# noqa`.
- Optional but recommended: run both through `pre-commit` with the hook versions
  pinned in `.pre-commit-config.yaml`, and the same checks again in CI.

### Documentation

- Every public module, class, function, and method gets a docstring in
  `{{DOCSTRING STYLE — e.g. Google or NumPy style}}`: what it does, each argument and
  the return value (don't repeat the type hints, do say what `None` means), the
  exceptions a caller should expect, units where relevant, and thread/async-safety
  when it isn't obvious. Ruff's `D` rules, set to that convention, enforce presence and
  shape.
