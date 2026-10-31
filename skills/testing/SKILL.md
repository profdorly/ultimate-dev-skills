---
name: testing
description: Use when writing, running, or fixing automated tests, including tests for new code, regression tests, one test or a whole suite, failing or flaky tests, snapshots and golden files, mocks, and coverage. Runs tests non-interactively, shows each new test can fail, keeps tests deterministic, and never deletes, skips, or weakens a test to get a green run.
license: MIT
metadata:
  author: profdorly
  version: "0.1.0"
---

# Testing

A test is evidence that code behaves as intended. It only counts if it can fail, fails for the right reason, and gives the same result every time it runs. A green suite reached by weakening tests is worse than a red one, because it tells everyone the code works when nobody knows.

## Rules (non-negotiable)

1. **Never make tests pass by weakening them.** That rules out deleting a test; adding `skip`, `xfail`, `.todo`, or `t.Skip`; leaving `.only` or `fit` behind; commenting out assertions; swapping an exact assertion for a vague one (`toBeTruthy`, `assert result`); widening tolerances; catching the error inside the test; changing expected values to match new output without showing the new output is correct; raising timeouts; and adding retries. If a test is wrong, show why and ask.
2. **Don't bend production code to the test.** No `if (process.env.NODE_ENV === 'test')` branches, test-only parameters, or hooks that exist only so a test can pass. Inject the dependency instead (a clock, an HTTP client, a repository).
3. **Snapshot and golden-file updates are reviewed changes.** Read the diff of every snapshot you update and tie each change to the task. Never bulk-update (`-u`, `--update-snapshots`, `--snapshot-update`, `cargo insta accept`) to clear failures.
4. **Every new test must have failed once.** Watch it fail before the fix, or break the code on purpose and watch it fail, with a message about the behavior rather than an import or syntax error. A test that has never failed hasn't been shown to test anything.
5. **Run tests non-interactively.** No watch modes, no interactive UIs, no reporters that open a browser and wait. Every test command you run must exit on its own.
6. **Tests never touch real external systems.** No production or shared databases, real payment or email providers, real user data, or the open internet, unless the project's suite is explicitly built for it (integration tests behind an opt-in flag, with credentials the user provides).
7. **Report results exactly.** State what ran, how many tests passed, failed, and were skipped, and what you couldn't run. Never say "all tests pass" after running a subset.
8. **Use what the project uses.** Don't add a test framework, plugin, or helper library without asking.

## Orient first

- **Find the real test command.** Look at what CI runs (`.github/workflows/`, `.gitlab-ci.yml`, and the like), then `package.json` scripts, `Makefile`, `justfile`, `tox.ini`/`noxfile.py`, `pyproject.toml`, `Cargo.toml`, `go.mod`, and `CONTRIBUTING.md`. Prefer the project's wrapper (`npm test --`, `make test`, `uv run pytest`, `bundle exec rspec`) so the versions, environment, and setup match.
- **Find the framework and its config:** `jest.config.*`, `vitest.config.*` or the `test` key in `vite.config.*`, `pytest.ini`/`pyproject.toml`/`conftest.py`, `playwright.config.*`, `.rspec`, `phpunit.xml`.
- **Learn the conventions** from neighboring tests: file locations and names, fixtures and factories, how they mock, how they set up the database, how they mark slow or integration tests.
- **Take a baseline.** Run the relevant tests once before you change anything. Failures that exist before your change aren't yours: record them so you neither chase them nor get blamed for them.

How to run one file or one test in each runner, and the flags that keep each one non-interactive, are in [references/running-tests.md](references/running-tests.md). The traps that catch agents most often:

- `vitest` alone starts watch mode in a terminal; use `vitest run`.
- `jest --watch` never exits; pass `--watchAll=false` if a script adds watch mode.
- Playwright's HTML reporter serves the report and waits when a test fails; set `PLAYWRIGHT_HTML_OPEN=never` or pass `--reporter=line`.
- Go caches passing results; use `-count=1` to force a real run.
- A name filter that matches nothing often exits 0: `go test -run`, `jest -t`, and `vitest run -t` all report success with zero tests run (pytest exits 5 instead). Check the count of tests that actually ran.

## Writing tests

- **Test behavior through the public interface,** the way callers use the code, not private methods or internal call order. Tests tied to implementation details break on every refactor and still miss real bugs.
- **One behavior per test, and say it in the name:** `test_rejects_expired_token`, `it('returns 404 for an unknown task id')`.
- **Arrange, act, assert,** with the minimum setup. Use the project's fixtures and factories instead of building large objects by hand.
- **Assert exact outcomes:** `assert result == {"id": 7, "status": "done"}` rather than `assert result`. Check the error type and message for failures, not only that something was raised.
- **Cover what breaks:** the main path, boundaries (zero, one, the maximum, empty, off-by-one), invalid input and error paths, and the specific case from the bug report.
- **Keep it deterministic.** Control time, randomness, the network, the file system, environment variables, and concurrency. The tools for each language are in [references/writing-tests.md](references/writing-tests.md).
- **Don't sleep.** Wait for a condition: await the promise, poll with a deadline, or use the framework's waiting assertions (Playwright's `await expect(locator).toHaveText(...)`).
- **Mock at the boundary,** for things you don't control (HTTP services, the clock, payment gateways, email). Don't mock the unit under test or its simple collaborators. Assert on a mock's calls only when the call is the behavior (an email was sent, a charge was created). A test made of mocks passes while the real code is broken.
- **Don't compute the expected value with the code under test** or a copy of its logic. Write the expected value out literally, or derive it independently.
- **No secrets or real personal data in fixtures.** Use obviously fake values.

## Fixing a failing test

1. **Read the failure:** the assertion diff, the stack trace, and the test's code.
2. **Run that test alone.** If it passes alone but fails in the suite, the problem is shared state or test order; see [references/flaky-tests.md](references/flaky-tests.md).
3. **Decide which side is wrong, with evidence:** the spec or issue, the commit that changed the behavior, the other tests, the docs.
   - The code is wrong: fix the code. The debugging skill, if installed, has the process.
   - The behavior changed on purpose, as part of the current task: update the test and say so in your report, citing the requirement.
   - It's unclear: ask. Don't pick whichever side makes the suite green.
4. **The environment is wrong** (a missing service, environment variable, or tool version): fix the setup or report it. Don't skip the test.

## Snapshots and golden files

1. Run the failing snapshot test and read the diff.
2. Decide, change by change, whether the task intended it. An unexplained difference is a bug until shown otherwise.
3. Update only the snapshots you reviewed, scoped to one file or test: `npx jest path -t 'name' -u`, `npx vitest run path -t 'name' -u`, `pytest path::test --snapshot-update` (syrupy), `npx playwright test path -g 'name' --update-snapshots`, `INSTA_UPDATE=always cargo test name` (insta), or the project's own flag for Go golden files (often `go test ./pkg -run TestName -update`).
4. Check the result with `git --no-pager diff -- <snapshot files>` and mention it in your report.
5. Read new snapshots too. A snapshot of an error page, `undefined`, or an empty list records a bug as the expected result.

## Coverage

Coverage shows which lines ran, not which behaviors are checked. Use it to find untested branches in the code you changed, not as a score to raise.

- Commands: `pytest --cov=<pkg> --cov-report=term-missing` (pytest-cov), `npx vitest run --coverage` (needs a coverage provider package), `npx jest --coverage`, `go test -coverprofile=cover.out ./... && go tool cover -func=cover.out`, `cargo llvm-cov` (cargo-llvm-cov).
- Never write tests without assertions to raise coverage, and never lower thresholds in the config to get past a gate.

## Stop and ask when

- A test's expected value seems wrong, or you'd need to delete, skip, or loosen a test.
- Making a test pass seems to require changing production behavior that the task didn't mention.
- Snapshot changes include anything you can't explain.
- Running the tests needs a real external service, credentials, a paid API, or data you don't have.
- The suite was already failing before your change and the failures block you from verifying your work.
- A test is flaky and the fix is outside the task.

## Final report

```
Tests:     added test_refund_cannot_exceed_charge (failed before the fix: refunded 500 of 100)
Ran:       uv run pytest tests/billing (63 passed, 0 failed, 2 skipped: pre-existing @slow marks)
Not run:   tests/integration (needs STRIPE_TEST_KEY, not set here)
Snapshots: none changed
Baseline:  tests/api/test_export.py::test_csv was already failing before my change (unrelated)
```

## Tempting shortcuts

| Shortcut | Why not |
|---|---|
| "The test is outdated, I'll update the expected value." | Maybe. Or the code regressed. Find out which before you change the test. |
| "Skip it for now; we'll fix it later." | A skipped test stops guarding anything, and "later" rarely comes. Ask first. |
| "Run `-u` and the snapshots pass." | You just accepted every change, intended or not, as the new truth. |
| "Mock everything so it's fast." | Then the test checks the mocks. Mock only at the boundaries. |
| "It's flaky; add a retry." | Retries hide real races and real bugs. Find the source ([references/flaky-tests.md](references/flaky-tests.md)). |
| "I'll add a sleep so the element is there." | It's slow when it works and still fails on a slow machine. Wait for the condition. |
| "The suite passes, so I'm done." | Did your new test ever fail? A test that can't fail proves nothing. |
| "I ran the tests" (a subset). | Say exactly which ones. A partial run reported as complete misleads everyone. |

## References

| Situation | Read |
|---|---|
| Running one file or one test in each framework, non-interactive flags, exit codes, test output | [references/running-tests.md](references/running-tests.md) |
| Controlling time, randomness, network, files, and environment; mocks and fakes; test databases; testing legacy code | [references/writing-tests.md](references/writing-tests.md) |
| A test that fails sometimes, passes alone but fails in the suite, or fails only in CI | [references/flaky-tests.md](references/flaky-tests.md) |
