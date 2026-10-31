# Flaky tests

A flaky test passes and fails on the same code. Either the test or the code has a real nondeterminism problem, and production code shares it more often than people expect. Find the source; don't retry it away.

## 1. Confirm and measure

Run the single test many times and stop at the first failure:

| Runner | Repeat command |
|---|---|
| Go | `go test ./pkg -run '^TestName$' -count=100 -failfast` |
| Playwright | `npx playwright test path -g 'name' --repeat-each=50 --reporter=line` |
| pytest | `pytest path::test_name --count=50 -x` with pytest-repeat installed, otherwise the loop below |
| Anything | the shell loop below |

```bash
for i in $(seq 1 50); do
  <single-test command> > "${TMPDIR:-/tmp}/flaky.log" 2>&1 || { echo "failed on run $i"; break; }
done
```

Then run it the same number of times inside the full suite or file, since many flakes only appear next to other tests.

With no failures in n runs, you can claim with about 95% confidence that the failure rate is below 3/n. A test that failed once in 20 runs needs about 60 clean runs before "fixed" is credible.

## 2. Find which factor matters

Change one thing at a time:

| Experiment | How | If it changes the result |
|---|---|---|
| Alone vs in the suite | Run the single test, then the whole file, then the suite | Shared state or test order |
| Fixed vs random order | pytest-randomly shuffles by default (`-p no:randomly` turns it off, `--randomly-seed=<n>` repeats an order); `go test -shuffle=on` (prints the seed; rerun with `-shuffle=<seed>`); `npx jest --randomize --seed=<n>`; `npx vitest run --sequence.shuffle` | Order dependence |
| Serial vs parallel | `pytest -n 0`, `npx jest --runInBand`, `npx vitest run --no-file-parallelism`, `go test -p 1 -parallel 1`, `cargo test -- --test-threads=1` | Shared resources: ports, files, database rows, globals |
| Timezone | `TZ=UTC`, `TZ=America/Sao_Paulo`, `TZ=Pacific/Kiritimati` (UTC+14) | Time zone or date math |
| Date | Freeze time near a month end, a daylight-saving change, Feb 29, or Dec 31 | Date boundaries |
| Load | Run while the machine is busy, or with fewer CPUs in a container | Races and timeouts |
| Seed | Print and fix the random seed | Unseeded randomness |

## 3. Fix the cause

| Cause | Signs | Fix |
|---|---|---|
| Shared state | Passes alone, fails in the suite | Fresh fixtures per test; reset globals, singletons, and module caches in teardown; never depend on rows another test made |
| Test order | Fails only in some orders | Same as shared state; make each test set up everything it needs |
| Time | Fails at certain hours, dates, or zones | Fake or inject the clock; compare in UTC; don't assert on `now()` |
| Unawaited async work | Errors appear after the test "passes", or in the next test | Await everything; close servers, clients, and timers in teardown |
| Fixed waits | `sleep`, `waitForTimeout`, or `setTimeout` in the test | Wait for the condition with a deadline |
| Shared ports, files, or names | "address already in use", file collisions | Bind port 0 and read the assigned port; per-test temp dirs; unique names |
| Randomness | Different data each run | Seed it, and print the seed on failure |
| Real network | Fails when a service is slow or down | Mock at the HTTP boundary |
| Resource limits | Fails only in CI or under parallelism | Lower parallelism for that suite, raise limits, or reduce what the test allocates |
| UI timing | Element not found, detached, or covered | Use the framework's auto-waiting locators and assertions; disable animations in tests; wait for network idle only when necessary |
| A real race in the code | The production code has the same window | Fix the code; the test found a bug |

## 4. What not to do

- **Retries as a fix** (`jest.retryTimes`, `@flaky`, pytest-rerunfailures, Playwright `retries`, CI "rerun failed jobs") hide the problem and also hide real regressions that happen to be intermittent. A retry is acceptable only as a stopgap the user approves, with a tracking issue named in a comment.
- **Quarantining** (skipping or moving the test to a non-blocking job) needs the user's approval and a tracking issue, the same as deleting it would.
- **Raising the timeout** without finding out why the operation got slow.
- **Declaring it fixed after a few green runs.** Use the numbers in step 1.

## 5. Report

```
Flaky:    tests/api/test_jobs.py::test_retry_backoff failed 3 of 50 runs
Factor:   fails only when run after test_cancel (fixed order reproduces it 50 of 50)
Cause:    test_cancel leaves a scheduled job in the module-level queue
Fix:      queue fixture now created per test; teardown drains it
Verified: 200 runs in the full file, 0 failures; full suite passed under 5 random orders (pytest-randomly)
```
