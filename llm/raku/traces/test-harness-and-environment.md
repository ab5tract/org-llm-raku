# Running and testing this plugin: environment gotchas

Practical, load-bearing knowledge for getting tests to run and trusting their
results. Several of these will waste an hour if you don't know them up front.

## Run against the newest Rakudo you have, and do not pin

Tests spawn a real `raku` subprocess: to load CORE symbols, and -- for the RakuAST
viewer -- to build the AST whose spans, slots and deparse output the tests assert
against. If `raku` isn't on `PATH`, they fail in `setUp()`
(`CommaFixtureTestCase.suggestSdkHome()` -> "Found a raku in path" assertion, or a
symbol-load timeout).

**The newest Rakudo wins.** Locally that is the source build under `$RAKU_PREFIX`,
which is what the RakuAST work is developed against: its spans and deparse rules
depend on node-origin information that older releases do not report at all.

```bash
export PATH="$RAKU_PREFIX/bin:$PATH"
./gradlew test --rerun --tests "..."
```

Without a source build, take the newest release rakubrew offers. `rakubrew
list-available` prints them oldest-first, so the last entry is the one you want, and
the switch must happen in the *same* shell as gradle, because shell state does not
persist between tool calls:

```bash
eval "$(~/.rakubrew/bin/rakubrew init Zsh)"
rakubrew switch moar-<newest>          # e.g. moar-2026.08
./gradlew test --rerun --tests "..."
```

Three things go wrong silently here, all of them ending in a green build that proves
nothing:

- **The rakubrew version is `moar-2026.08`, not `2026.08`.** A bare `2026.08` prints
  "Sorry, '2026.08' not found. Did you mean: moar-2026.08" -- but through the
  `rakubrew` shell function that `init` installs it still returns success, so `&&`
  chains march on with the switch silently not applied. (Called directly as
  `~/.rakubrew/bin/rakubrew`, outside the hook, the same mistake exits 1.)
- **`--rerun` is not optional when you have changed only the environment.** `PATH` and
  the SDK are not declared inputs of the `test` task, so switching Rakudo leaves it
  `UP-TO-DATE`. `./gradlew test` then reports `BUILD SUCCESSFUL in 629ms` having
  executed zero tests. Check for `> Task :test UP-TO-DATE` before believing a result,
  and confirm the "N tests completed" line is present.
- **`suggestSdkHome()` takes the *first* `PATH` entry that looks like a Raku SDK
  home.** Prepend, never append -- otherwise the system Rakudo in `/usr/bin` wins and
  symbol-dependent assertions fail in ways that impersonate plugin bugs.

`~/.rakubrew/MODE` is `env` here, so rakubrew works by rewriting `PATH` from the shell
hook; there are no shims. A `PATH` that lacks the hook is the normal failure mode for
a non-interactive shell that never sourced the user's profile.

## A failing symbol-dependent assertion is a version question first

CORE.setting and RakuAST both genuinely differ between releases, so an expectation can
fail while the plugin is faithfully reporting whatever Rakudo told it. **Check
`raku -v` before you touch the expectation**, and confirm the difference at its source
-- ask Rakudo directly rather than inferring it from the test:

| | older | newer |
|---|---|---|
| `*%_` declared type (2026.03 -> 2026.08) | `Mu` | `Associative` |
| `RakuAST::StrLiteral` origin for `my $x = "cool";` (2026.03 -> 2026.08) | no span at all | span `cool`, drag span `"cool"` |
| `Mu.^find_method("perl").candidates[0].DEPRECATED` (2025.08 -> 2026.03) | *absent* | `raku` |
| `&open.candidates` (2025.08 -> 2026.03) | 1 -- `(IO(Any) $path, \|c)` | 2 -- `("-", \|c)`, `($path, \|c)` |

On 2025.08 `raku-core-symbols.raku` emits `perl` with no `x` (deprecation) key, so no
deprecation warning is possible; and `open` is not yet a multi with several candidates,
so an arity error reads "Not enough positional arguments" rather than enumerating
"No multi candidates match (...)".

**When they differ, move the expectation forward. Do not pin the environment back.**
This file used to advise the opposite, and the cost is on record twice over.

The `.perl` expectation was removed to get the suite green on 2025.08, and then the
environment was pinned to 2026.03, where `Mu.perl` *is* deprecated. The same test went
on failing with the comparison inverted -- the warning now reported as **extra** rather
than missing. Two fixes, each locally reasonable, that cancelled out.

Then the pin rotted in the other direction: held at 2026.03, all eight RakuAST viewer
tests failed, because 2026.03 does not report the node origins the viewer is built on
-- `StrLiteral` had no span, so the drag span came back as `= "cool"` and slots
resolved one level up (`initializer` where the test expected `expression`). Nothing was
wrong with the plugin. The fix was the environment, and the three `*%_` expectations
that had quietly encoded 2026.03 moved forward with it.

Where the varying part belongs to Rakudo rather than to the plugin, prefer an assertion
that does not pin it at all -- see the next section.

Note that the deprecation lives on the *candidate*, not on the proto (`is DEPRECATED`
is a `Method+{is-DEPRECATED}` mixin), which is why probing `$m.DEPRECATED` on the proto
reports nothing even where it is deprecated.

## Assertions built from CORE.setting text should not be pinned exactly

`CommaFixtureTestCase.checkHighlightingContains(vararg fragments)` asserts that the
rendered actual highlighting *contains* each fragment, for annotations whose full text
is assembled from the SDK's own signatures. `testCallArityMismatchAnnotating` uses it
for `open;`: the multi-candidate list is Rakudo's, not the plugin's, so pinning it
exactly pins a Rakudo release for no gain in coverage.

Keep a fragment that closes a span (e.g. `open</error>`) in the list, otherwise the
assertion passes even when the annotation vanishes entirely. Configure the source
*without* expectation markup when using it.

This is the right tool only when the varying part belongs to Rakudo. An annotation the
plugin composes itself should still be pinned exactly.


## ~~The `checkHighlighting()` pipeline is broken~~ / ~~use a checkpoint subset~~ — RESOLVED

**Both of these claims are obsolete. Do not act on them.** They were symptoms of one
harness bug, fixed in `test-harness-project-reuse.md`: the full suite now runs ~1100
tests in ~2 minutes, and `org.raku.comma.annotation.*` / `org.raku.comma.highlighting.*`
are as trustworthy as any other suite.

What was really happening: the light project was rebuilt for *every* test, so each test
respawned a ~4s `raku` CORE-symbols subprocess. Under that load a spawn would sometimes
fail and `getCoreSettingFile()` silently fell back to the stale bundled
`symbols/CORE.fallback` — so symbol-dependent highlighting assertions failed
non-deterministically, which read as "the pipeline is broken". The ~69-minute runtime
that motivated the "checkpoint subset" advice had the same single cause.

Still true and still worth doing: **direct-invocation inspection tests are a good
idea on their own merits** — faster and far more precise than a golden-highlighting
comparison. Invoke `provideVisitFunction` over the PSI and assert on
`ProblemsHolder.results` (see `MissingRoleMethodInspectionTest`,
`RedeclaredImportedSymbolInspectionTest`). For parsing, use the golden-PSI-tree
`RakuParsingTestCase` framework (see `parser-generated-lexer-architecture.md`).

## Logged errors are escalated to hard failures under test

All test JVMs run with `-Dintellij.testFramework.rethrow.logged.errors=true`. Any
`Logger.error(...)` becomes a `TestLoggerAssertionError`. So things a real IDE only
logs-and-continues (a background coroutine failing, a stub-index-during-stub-building
warning) become test failures. Two real examples handled this way:

- **CodeVision** (`c7beea6b`): the bundled Kotlin plugin's
  `KotlinScriptDefinitionCodeVisionProvider` fails a resource-bundle lookup in this
  sandbox and logs an error from a background coroutine during project startup. In
  real use it's harmless. `CommaFixtureTestCase.runBare()` scope-suppresses **only
  that one message** via `LoggedErrorProcessor` so genuine logged-error regressions
  still fail loudly. `runBare()` (not `runTestRunnable()`) is required because the
  coroutine can fire during `setUp()`, before the test body.
- **Stub-index-during-stub-building** — see `stub-building-index-queries.md`.

Pattern for suppressing exactly one known-benign message in a scratch test:
```kotlin
LoggedErrorProcessor.executeWith<Throwable>(object : LoggedErrorProcessor() {
    override fun processError(category, message, details, t) =
        if (message.contains("<known-benign-substring>")) Action.NONE
        else super.processError(category, message, details, t)
}) { /* body */ }
```

## Just run the full suite

`./gradlew test` is ~2 minutes for 1087 tests. There is no longer any reason to guess
at a "checkpoint subset" — run everything. See `test-harness-project-reuse.md` for why
it used to take over an hour.

## Tests that need a Raku module you don't have installed

`CommaFixtureTestCase.ensureModuleIsLoaded` probes the SDK (`raku -e 'use X'`, cached
per JVM) and aborts the test as `[SKIPPED]` when the module genuinely isn't installed,
instead of letting it fail later on a content assertion. `Cro::WebApp::Template`,
`Cro::WebApp::Form` and `OO::Monitors` are not installed on every dev machine, so
`GoToDeclarationTest`'s template-jump tests, `TraitCompletionTest.testAttributeTrait`
and `MethodCompletionTest.testMetaMethodCompletion` skip unless you `zef install` them.

A skip is deliberately *not* the same as a pass-with-no-symbols: if the module **is**
installed and symbol loading then fails, the test still fails. Grep the run log for
`[SKIPPED]` to see what didn't actually run.

Pass `required = false` when the test only needs the module *named* in the source it
operates on rather than resolved — `IntentionTest`'s two monitor cases rewrite code that
mentions `OO::Monitors` without ever looking a symbol up, and skipping those would have
been a silent loss of coverage. Those log `[NOTE]` instead and keep running.

## Scratch tests: write outputs to a randomized temp dir

Investigation/scratch tests should write to a fresh randomized directory under the
user temp dir (JVM `java.io.tmpdir`, the equivalent of Raku's `$*TMPDIR`), not a fixed
path, so repeated/concurrent runs don't clobber each other and echo the path so it's
discoverable:
```kotlin
val dir = Files.createTempDirectory("raku-scratch-<label>-").toFile()
File(dir, "out.txt").writeText(...); println("[scratch] wrote ${'$'}dir/out.txt")
```

## Running the sandbox IDE

`./gradlew runIde` launches a sandbox IDE with the plugin. Used to reproduce
user-facing symptoms end to end (the test suite is trustworthy again, but seeing it is
still seeing it). `.rakumod` module-file support depends on `<depends>com.intellij.modules.platform</depends>`
/ JCEF wiring in `plugin.xml` (a tab that opens and immediately closes is that
dependency missing — cf. commit `c2c45083` "Fix .rakumod loading by explicitly
depending on JCEF").
