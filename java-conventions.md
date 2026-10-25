<!-- Paste the body below into CLAUDE.md in place of the {{CONVENTIONS}} placeholder,
     and drop or demote this H2 so CLAUDE.md keeps a single `## Conventions` heading. -->

## Conventions — Java (Maven or Gradle / JUnit 5)

**Stack:** Java 17 (LTS) on Linux and macOS, built with `{{BUILD TOOL — Maven or Gradle}}`,
tested with JUnit 5 (Jupiter). Java 17 is the baseline for both the language and the
standard library. The project may move to Java 21 (LTS); where a Java 21 feature is a
clear improvement, it's listed under **Java 21 opt-in** below together with the Java 17
way to do the same thing. Until the baseline is actually raised, write the Java 17 form.
JDKs/platforms: `{{JDKS & PLATFORMS — e.g. Temurin 17 and 21 on Linux x86_64/aarch64 and macOS arm64}}`.

### Build & layout

- The Java level is set in **one place** and uses `--release`, not `-source`/`-target`:
  - Maven: `<maven.compiler.release>17</maven.compiler.release>` in the parent POM.
  - Gradle: `java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }` plus
    `tasks.withType(JavaCompile).configureEach { options.release = 17 }`.

  `--release 17` also rejects *library* APIs newer than 17, so building on a JDK 21
  still catches an accidental `List.getFirst()`. Don't use `-source`/`-target`; they
  don't do that.
- No preview features (`--enable-preview`) and no incubator modules, at either baseline.
- The build tool is pinned and committed: the Gradle wrapper (`gradlew`,
  `gradle/wrapper/`) or the Maven wrapper (`mvnw`, `.mvn/wrapper/`). Always invoke the
  wrapper, never a system-installed `mvn`/`gradle`. Maven projects also run the
  Enforcer plugin with `requireJavaVersion` and `requireMavenVersion`.
- Every dependency and plugin has an exact version — no ranges, `+`, `LATEST`, or
  `SNAPSHOT` of something this repo doesn't own. Versions live in one place
  (`<dependencyManagement>`/`<pluginManagement>` or a Gradle version catalog,
  `gradle/libs.versions.toml`).
- Standard layout: `src/main/java`, `src/main/resources`, `src/test/java`,
  `src/test/resources`. One top-level package root, `{{BASE PACKAGE — e.g. org.example.project}}`.
  `build/` and `target/` are never committed.
- Compiler warnings: `-Xlint:all` always on (minus specific categories, each with a
  comment saying why). Move to `-Werror` once the codebase is warning-clean, as a build
  property CI turns on. Don't silence a warning with `@SuppressWarnings` on a whole class;
  put it on the narrowest declaration, with a comment.
- No reaching into JDK internals: no `sun.*` / `com.sun.*` (other than the documented
  `com.sun.net.httpserver` etc.), no `--add-opens`/`--add-exports`, no
  `setAccessible(true)` on JDK classes. These break silently on JDK upgrades.

### Language & style

- Immutability by default: fields are `final` unless there's a reason; collections
  returned from a public API are unmodifiable (`List.of`, `List.copyOf`,
  `Collections.unmodifiableList`) or defensively copied.
- **Records** for plain value carriers. Validate and defensively copy in the compact
  constructor (`items = List.copyOf(items);`). A record holding an array or mutable
  object breaks value semantics — don't.
- **Sealed** interfaces/classes for closed hierarchies, with the permitted subtypes in
  the same package (or same file as nested types).
- Switch **expressions** (`case X -> ...`) over statement `switch` with fall-through.
  At Java 17, `switch` takes only enums, strings, and primitives/boxes — not type
  patterns. To dispatch on a sealed hierarchy, use an `instanceof` pattern chain
  (`if (s instanceof Circle c) ... else if ...`) ending in a `throw new
  AssertionError(s)` for the unreachable case, or a visitor. See Java 21 opt-in.
- Pattern matching for `instanceof` (`if (o instanceof Foo f)`) rather than test-then-cast.
- Text blocks (`"""`) for multi-line literals (SQL, JSON, expected output in tests).
- `var` only when the type is obvious from the right-hand side
  (`var list = new ArrayList<String>()`), never for a method call whose return type
  the reader must look up, and never for numeric literals.
- Null handling: public methods validate non-null parameters with
  `Objects.requireNonNull(x, "x")`. `Optional` is for return values that may be
  absent — never a field, parameter, or collection element, and never `null` itself.
  Return empty collections, not `null`. Nullness annotations:
  `{{NULLNESS ANNOTATIONS — e.g. JSpecify @Nullable / @NullMarked, or none}}`.
- `equals` and `hashCode` are overridden together, or not at all (records do both).
  `compareTo` is consistent with `equals`, or the Javadoc says it isn't.
- Exceptions:
  - Checked exceptions for recoverable conditions the caller is expected to handle
    (I/O, malformed input); unchecked for programming errors (bad arguments, broken
    invariants). Don't use exceptions for routine control flow (not-found,
    end-of-input).
  - Never catch `Exception`/`Throwable` except at a top-level boundary (`main`, a
    request handler, a thread's run loop), and there, log it with the stack trace.
  - Never swallow an exception. When wrapping, pass the cause (`new FooException(msg, e)`).
  - Catching `InterruptedException` means re-asserting
    `Thread.currentThread().interrupt()` or re-throwing — never ignoring it.
  - Every `Closeable`/`AutoCloseable` that the code opens is closed by
    try-with-resources. That includes `Files.lines`, `Files.list`, `Files.walk`
    streams.
- `java.time` only for dates and times: no `java.util.Date`, `Calendar`, or
  `SimpleDateFormat`. Use `Instant` for timestamps, and pass a `Clock` into code that
  reads the time so tests can fix it.
- `BigDecimal` (constructed from a `String`, not a `double`) for money or any decimal
  value that must be exact.
- Concurrency uses `java.util.concurrent` (executors, `CompletableFuture`, concurrent
  collections, atomics) rather than raw `Thread`, `wait`/`notify`. Every
  `ExecutorService` the code creates is shut down in a `finally` (at Java 17 it isn't
  `AutoCloseable`). Shared mutable state is documented with the lock or mechanism that
  guards it.
- Off-limits: Java serialization (`Serializable`, `ObjectInputStream`) for anything
  new; `finalize()`; `Thread.stop`/`suspend`; the Security Manager (deprecated for
  removal in 17); raw types; `System.exit` outside `main`.
- Logging through `{{LOGGING API — e.g. SLF4J, or java.lang.System.Logger}}`. Library
  code never writes to `System.out`/`System.err` and never calls `printStackTrace()`.
  Use parameterized messages (`log.debug("x={}", x)`), not string concatenation.
- APIs newer than Java 17 are off-limits at the 17 baseline (the compiler enforces this
  via `--release`; don't route around it with reflection or a multi-release JAR). If
  one is truly needed before the baseline moves, that's a decision to record as an
  ADR, not a quiet workaround.

### Java 21 opt-in

Raising the baseline is one change: set `release` (and the Gradle toolchain) to `21`,
update the CI JDK matrix, and record an ADR. Nothing below is allowed until that's
done. After it is, prefer the right-hand column in new code; existing Java 17 code is
converted only when touched for another reason, or as a `TASK-###`.

| Need | Java 17 (baseline) | Java 21 option |
|---|---|---|
| Dispatch on a sealed type | `instanceof` pattern chain ending in `throw new AssertionError(x)`, or a visitor | Pattern-matching `switch` with **no `default`**, so adding a subtype is a compile error at every switch |
| Destructure a record | Accessor calls after an `instanceof` pattern | Record patterns: `if (p instanceof Point(var x, var y))`, and nested in `case` labels |
| Guarded cases | `if`/`else` inside the branch | `case Shape s when s.area() > 0 ->` |
| `null` in a switch | Check for `null` before the `switch` | `case null ->` (otherwise `switch` still throws `NullPointerException`) |
| First/last/reversed element | `list.get(0)`, `list.get(list.size() - 1)`, `deque.peekFirst()`, `new ArrayList<>(l)` + `Collections.reverse` | `SequencedCollection`: `getFirst()`, `getLast()`, `reversed()` (also `SequencedMap.firstEntry()` etc.) |
| Many blocking I/O tasks | Bounded `Executors.newFixedThreadPool(n)`, shut down in `finally` | `try (var ex = Executors.newVirtualThreadPerTaskExecutor()) { ... }` — `ExecutorService` is `AutoCloseable` in 21. Don't pool virtual threads, and avoid holding a `synchronized` monitor across blocking calls on them (it pins the carrier thread in 21); use `ReentrantLock` there |
| Clamp a number | `Math.max(lo, Math.min(hi, v))` | `Math.clamp(v, lo, hi)` |
| Repeat into a builder | `sb.append(s.repeat(n))` | `sb.repeat(s, n)` |

Still off-limits at 21: preview features (string templates, unnamed patterns `_`,
unnamed classes, scoped values, structured concurrency).

### Portability (Linux & macOS)

- Always name the charset: `StandardCharsets.UTF_8` on every `String`↔`byte[]`
  conversion, `Reader`/`Writer`, `Files.readString`/`newBufferedReader`, etc. The
  default charset is platform-dependent in Java 17 (it becomes UTF-8 in 18+), so code
  that relies on it behaves differently on different machines and JDKs.
- Always name the locale for anything machine-readable: `toLowerCase(Locale.ROOT)`,
  `String.format(Locale.ROOT, ...)`. With the default locale, `"I".toLowerCase()` is
  not `"i"` in Turkish, and `%.2f` prints a comma in much of Europe.
- Paths via `java.nio.file.Path`/`Files`, built with `resolve`, never by concatenating
  `"/"`. Don't use `java.io.File` in new code.
- macOS filesystems are usually case-insensitive and Linux's are case-sensitive: class,
  package, resource, and file names must match case exactly. A resource that loads on
  a Mac can be missing on Linux.
- Write `\n` explicitly in machine-readable output; `%n` and `println` emit the
  platform line separator. Baseline comparisons must not depend on either.
- Shelling out goes through `ProcessBuilder` with an argument list, never a single
  command string through `sh -c`. Don't assume GNU tools are what's on the `PATH`.
- Native code (JNI/JNA/FFM): `{{NATIVE CODE POLICY — e.g. none}}`. If any is added,
  each OS/arch's library is loaded through one class, exercised by a test.

### Testing

- `{{TEST COMMAND — ./mvnw verify or ./gradlew check}}` is the single command that
  builds and runs everything, including the static-analysis checks below. A test the
  build doesn't run doesn't exist.
- Unit tests use JUnit 5 (Jupiter) in `src/test/java`, mirroring the package of the
  class under test. Assertions: `{{ASSERTIONS — JUnit's Assertions, or AssertJ}}`;
  pick one and use it everywhere. Mocking (`{{MOCKING — e.g. Mockito, or none}}`) is
  for collaborators the test can't run for real — not for value objects or the class
  under test.
- **Baseline tests** — where the output is a text artifact (serialized response,
  generated metadata, CLI output) — compare actual output to a checked-in `*.baseline`
  file under `src/test/resources`, fail on any difference, and print a unified diff.
  Baselines are read and written as UTF-8 with `\n` line endings. They change only
  through one deliberate update mode that rewrites them instead of comparing:
  `{{UPDATE MECHANISM — e.g. -Dupdate.baselines=true}}`, which writes to the source
  tree, not to `target/`/`build/`. Never hand-edit a baseline. Update mode is off by
  default and never on in CI. Review `git diff` on baseline files like source code; a
  baseline change in a PR needs a reason.
- Integration tests that need real files use JUnit's `@TempDir`, never a fixed path or
  the working directory. Tests that start servers bind to port `0` and read back the
  port.
- Tests are deterministic and independent: no reliance on test order
  (`@TestMethodOrder` only with a comment saying why), wall-clock time (inject a fixed
  `Clock`), network, default locale/charset/time zone, or shared static state — so
  JUnit's parallel execution is always safe. Fixed seeds only.
- New logic isn't done without a test. A bug fix isn't done without a regression test
  that fails before the fix and passes after.
- Coverage is measured with JaCoCo in CI: `{{COVERAGE GATE — e.g. report only, or a
  minimum line/branch ratio}}`. A gate is a floor, not a goal; don't write tests that
  only execute lines.

### Static analysis & formatting

- `{{FORMATTER — e.g. Spotless with google-java-format, or palantir-java-format}}` is
  the formatting source of truth, enforced by the build (`spotlessCheck`/`spotless:check`).
  Don't hand-format around it.
- `{{ANALYZERS — e.g. Error Prone, SpotBugs, Checkstyle}}` runs clean as part of the
  build before a change is considered finished, or a new finding is suppressed at the
  narrowest scope with a comment saying why.

### Documentation

- Every public type and public/protected method gets a Javadoc comment: what it does,
  `@param`/`@return` including whether `null` is accepted or returned, `@throws` for
  every checked exception and any unchecked one a caller should expect, units where
  relevant, and thread-safety when it isn't obvious. Javadoc builds with
  `-Xdoclint:all` (minus `missing` for non-public code) without warnings.
