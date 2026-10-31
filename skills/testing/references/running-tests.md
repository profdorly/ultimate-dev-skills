# Running tests

Prefer the project's own wrapper when there is one (`npm test -- <args>`, `pnpm test <args>`, `make test`, `uv run pytest`, `poetry run pytest`, `bundle exec rspec`). It sets up the right versions, environment variables, and services. Pass extra arguments through it rather than calling the runner directly with different settings.

## One file, one test, no watch mode

| Runner | One file | One test | Notes |
|---|---|---|---|
| pytest | `pytest tests/test_x.py` | `pytest tests/test_x.py::test_name`, `pytest tests/test_x.py::TestClass::test_name`, or `-k 'name'` | `-x` stops at the first failure, `-q` shortens output, `--lf` reruns the last failures. With pytest-xdist, `-n 0` runs serially. |
| unittest | `python -m unittest tests.test_x` | `python -m unittest tests.test_x.TestCase.test_name` | |
| Django | `python manage.py test app.tests.test_x` | `python manage.py test app.tests.test_x.TestCase.test_name` | `--keepdb` reuses the test database between runs. |
| Jest | `npx jest path/to/file.test.ts` | `npx jest path/to/file.test.ts -t 'name'` | `--watchAll=false` overrides a script that adds `--watch`. `--ci` makes missing snapshots fail instead of being written. |
| Vitest | `npx vitest run path/to/file.test.ts` | `npx vitest run path -t 'name'` | Always `vitest run`; bare `vitest` watches in a terminal. |
| Mocha | `npx mocha path/to/file.spec.js` | `npx mocha path --grep 'name'` | |
| Node test runner | `node --test path/to/file.test.js` | `node --test --test-name-pattern='name' path` | |
| Bun | `bun test path` | `bun test path -t 'name'` | |
| Playwright | `npx playwright test path` | `npx playwright test path -g 'name'` | `--reporter=line`, `PLAYWRIGHT_HTML_OPEN=never`, `--project=chromium`. Never `--ui` or `--debug`. |
| Cypress | `npx cypress run --spec path` | Filter with the project's grep plugin if it has one | Never `cypress open`. |
| Go | `go test ./pkg/...` | `go test ./pkg -run '^TestName$'`; subtests: `-run '^TestName$/^case_name$'` | `-count=1` skips the result cache, `-race` enables the race detector, `-v` prints each test. |
| Cargo | `cargo test --test <integration_file>` | `cargo test <name> -- --exact` | `-- --nocapture` shows prints. Use `cargo nextest run` if the project does. |
| RSpec | `bundle exec rspec spec/x_spec.rb` | `bundle exec rspec spec/x_spec.rb:42` | |
| Rails / Minitest | `bin/rails test test/x_test.rb` | `bin/rails test test/x_test.rb:42` | |
| PHPUnit | `vendor/bin/phpunit tests/XTest.php` | `vendor/bin/phpunit --filter testName` | Use `vendor/bin/pest` for Pest projects. |
| .NET | `dotnet test path/To.Tests.csproj` | `dotnet test --filter "FullyQualifiedName~Namespace.Class.Method"` | |
| Maven | `mvn -q test -Dtest=ClassTest` | `mvn -q test -Dtest='ClassTest#method'` | In multi-module builds add `-Dsurefire.failIfNoSpecifiedTests=false`. |
| Gradle | `./gradlew test --tests 'pkg.ClassTest'` | `./gradlew test --tests 'pkg.ClassTest.method'` | Gradle skips up-to-date tests; `--rerun` forces them (Gradle 7.6 and later). |
| Elixir | `mix test test/x_test.exs` | `mix test test/x_test.exs:42` | |
| Dart / Flutter | `dart test path` or `flutter test path` | `dart test path -N 'name'` | |
| Swift | `swift test` | `swift test --filter 'Module.Class/testName'` | |

## Keeping it non-interactive

- Wrap long or hang-prone runs in a deadline: `timeout 600 <command>` where `timeout` exists (`command -v timeout`), otherwise `perl -e 'alarm shift; exec @ARGV' 600 <command>`.
- Some tools switch to CI behavior when `CI` is set: no watch mode and no prompts, but also no snapshot writing (Jest), `forbidOnly` and retries (typical Playwright configs), and different reporters. Prefer the explicit flags above. If you do set `CI=1`, say so in your report.
- Turn off colors when parsing output: `NO_COLOR=1`, or `FORCE_COLOR=0` for tools based on chalk.
- Never start an interactive UI or debugger: `vitest --ui`, `playwright test --ui`, `cypress open`, `pytest --pdb`, `node --inspect-brk`.

## Did the test actually run?

Filters that match nothing are a common way to get a misleading green result:

| Runner | What happens when nothing matches |
|---|---|
| pytest | Exits 5, also when `-k` deselects every test. Treat it as a mistake in your command, not a pass. |
| Go | Exits 0. The summary line ends with `[no tests to run]`; with `-v`, Go also prints `testing: warning: no tests to run`. Check for either. |
| Jest | A path that matches no file exits 1 ("No tests found") unless `--passWithNoTests` is set. A `-t` name that matches nothing marks every test skipped and exits 0. |
| Vitest | Same as Jest: no matching file exits 1 ("No test files found"); a `-t` name that matches nothing skips everything and exits 0. |
| Cargo | Runs zero tests and exits 0; read the `test result:` lines. |

After every run, read the totals line and confirm the test you meant is in it, with `-v`, `--reporter=verbose`, or the runner's equivalent if necessary.

## Exit codes worth knowing

- pytest: 0 all passed, 1 some failed, 2 interrupted, 3 internal error, 4 usage error, 5 no tests collected.
- `gh pr checks`, if you're reading CI results: 8 means checks are still pending.
- A runner killed by a signal exits with 128 plus the signal number: 137 is SIGKILL (often the out-of-memory killer), 143 is SIGTERM. The deadline wrappers above exit with their own codes when time runs out: 124 for `timeout`, 142 (SIGALRM) for the `perl` alarm.

## Reading the output

- Long logs: save them to a file (`<command> > "${TMPDIR:-/tmp}/test.log" 2>&1`), then read the summary and the first failure. The first failure usually causes the rest.
- Use the runner's own options to shrink output, such as `-q`, `--reporter=dot`, or `--reporter=line`, rather than piping through `tail`, which can cut off the failure details you need.
- Parse results from machine-readable output when you need exact counts: `pytest --junitxml=<file>`, `npx jest --json --outputFile=<file>`, `npx vitest run --reporter=json --outputFile=<file>`, `go test -json`.

## Services the tests need

- Look for how the project starts them: `docker compose` files, `Makefile` targets, `scripts/test-setup*`, testcontainers in the test code, or CI service definitions.
- Starting a local container for tests is usually fine; say what you started and stop it afterwards. Ask before deleting volumes, resetting a database that might hold the user's local data, or pointing tests at anything that isn't local.
- If tests need credentials or external services you don't have, report which tests you couldn't run and why. Don't skip them in code.
