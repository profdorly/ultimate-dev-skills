# Hard cases

## When you can't reproduce it

Compare the failing environment with yours, one category at a time, before reading more code:

- **Versions:** language runtime, OS and architecture, dependencies (was the lockfile honored, `npm ci` versus `npm install`?), database and service versions, browser.
- **Configuration:** environment variables, `.env` files, feature flags, CLI flags, config files that exist only on one machine.
- **Locale and time:** `TZ`, `LANG`/`LC_ALL`, the current date (month end, daylight-saving changes, leap days, year boundaries), clock skew between machines.
- **File system:** case sensitivity (macOS is usually case-insensitive, Linux is not), line endings (CRLF), path separators, permissions and umask, symlinks, maximum path length.
- **Data:** the specific record or input; empty, huge, Unicode, or null values; rows written by an older version of the code.
- **State:** caches, stale build output (`dist/`, `__pycache__`, `.next`, incremental compiler caches), leftover database rows from earlier tests, files from earlier runs.
- **Timing and load:** concurrency, parallel test workers, slower disks or networks, cold caches, CPU and memory limits.
- **Network:** proxies, DNS, TLS certificates, firewalls, sandboxed runners, rate limits.

Ask the reporter for whatever narrows it down: the exact version or commit, full logs, request IDs, timestamps, and the input that failed. If it still won't reproduce, report what you checked and propose targeted logging at the suspected boundary instead of a speculative fix. If you make a defensive change anyway, label it as unverified.

## Intermittent failures

- **Measure the failure rate first.** Run the narrowest reproduction in a loop and stop at the first failure:

  ```bash
  for i in $(seq 1 50); do
    <command> > "${TMPDIR:-/tmp}/run.log" 2>&1 || { echo "failed on run $i"; break; }
  done
  ```

- **Know how many clean runs prove a fix.** With no failures in n runs, you can claim with about 95% confidence that the failure rate is below 3/n. To show that a 1-in-20 failure is gone, you need about 60 clean runs.
- **Vary one factor at a time:** serial versus parallel, a fixed versus a random test order, a fixed seed, more load, a different timezone.
- **Usual causes:** state shared between tests, test order, the current time, unseeded randomness, real network calls, unawaited async work, missing locks, exhausted resources (ports, file handles, connection pools), and eventual consistency in external systems.
- **Never fix with a sleep.** Wait for the condition you need (poll with a deadline, await the event, use the test framework's waiting assertions).

For flaky tests specifically, the testing skill has a fuller recipe.

## Only in CI

- Read the workflow file and run the exact commands CI runs, with the same flags, not your usual shortcut.
- Expect these differences: a clean checkout with no ignored files or caches; `CI=true`, which changes the behavior of many tools (Jest snapshots, Playwright retries and reporters); a different OS, architecture, or timezone (usually UTC); fewer CPUs and less memory; no TTY; missing secrets on pull requests from forks; service containers instead of local services.
- Reproduce a clean checkout in a container without touching the user's working tree. This builds only committed content (`HEAD`), so commit or apply your change in a throwaway worktree first if you need to test it:

  ```bash
  git archive --format=tar HEAD | docker run --rm -i -e CI=true -w /work node:22 \
    sh -c 'tar -xf - && npm ci && npm test'
  ```

  Use the image and commands from the workflow file. Don't bind-mount the user's working directory read-write into a container that installs dependencies: it would overwrite their local `node_modules`, virtualenv, or build output with Linux builds.
- To read the failing logs from GitHub Actions, see the pull-requests skill (`gh run view <run-id> --log-failed`).

## Only in production

- Don't debug against production without the user's approval. Start read-only: logs, metrics, traces, the error tracker. Correlate by request ID and timestamp.
- Reproduce locally with the same versions, config, and data shape (anonymized data, never a copy of customer records without approval).
- Usual causes: config or feature-flag differences, data volume (an N+1 query that only hurts at scale), rows written by old versions, real concurrency, timeouts between services, and resource limits.
- Adding logging to production is a deploy. Propose it; don't do it yourself.

## Hangs and deadlocks

- Run anything that might hang with a deadline: `timeout 120 <command>` (GNU coreutils and recent macOS have `timeout`; check `command -v timeout`). Without it, `perl -e 'alarm shift; exec @ARGV' 120 <command>` works on any system with Perl.
- Take a stack dump of the stuck process instead of guessing; see the table in [techniques.md](techniques.md#debuggers-without-an-interactive-terminal).
- Usual causes: a process waiting for input on stdin (an interactive prompt), locks taken in different orders, awaiting a promise or future that never resolves, blocking I/O on an event loop, a subprocess whose output pipe nobody reads, an exhausted connection pool, and retry loops without a limit.

## Memory growth

- Prove growth with a series of measurements, not one number: sample RSS while repeating the operation (`ps -o rss= -p <pid>`), and check whether it levels off after garbage collection.
- Compare heap snapshots taken before and after repeating the operation:
  - Python: `tracemalloc.start()`, then `snapshot.compare_to(previous, 'lineno')`.
  - Node: `node --heapsnapshot-signal=SIGUSR2 app.js`, then `kill -USR2 <pid>` twice and compare the snapshots; or `node --heap-prof`.
  - Go: `go tool pprof -top http://localhost:6060/debug/pprof/heap` when the program serves `net/http/pprof`.
  - JVM: `jcmd <pid> GC.class_histogram`, or `jcmd <pid> GC.heap_dump <file>`.
- Usual causes: caches and maps without a size limit, event listeners or subscriptions never removed, global lists, closures holding large objects, and goroutines or tasks that never finish.

## Races and concurrency

- Use the race detectors you have: `go test -race ./...`; ThreadSanitizer for C and C++ (`-fsanitize=thread` with clang or gcc); for Rust on nightly, `RUSTFLAGS=-Zsanitizer=thread RUSTDOCFLAGS=-Zsanitizer=thread cargo +nightly test -Zbuild-std --target <host-triple>` (supported targets only), and Miri (`cargo +nightly miri test`) for undefined behavior.
- To confirm a suspected race, widen the window: put a temporary delay inside the suspected critical section in a test and check that the failure becomes reliable. Remove the delay afterwards.
- Fix with real synchronization (locks, atomics, transactions, unique constraints) or by removing the shared state. Never with a sleep.

## Wrong results with no error

- Find the first point where the data goes wrong: log intermediate values for a tiny input and compare them with a result you worked out by hand.
- Usual causes: off-by-one errors; integer division and overflow; floating-point money; timezones and daylight saving; text encoding and Unicode normalization (NFC versus NFD); mutable default arguments in Python; aliasing and shallow copies; comparing strings with numbers (`"10" < "9"` is true in JavaScript); locale-dependent sorting; `null` versus `undefined` versus a missing key; stale cached values; mixed units (seconds versus milliseconds).

## Performance regressions

- Measure before and after with the same input on the same machine, and use the median of several runs.
- Profile instead of guessing: `py-spy record -o profile.svg -- python app.py` or `python -m cProfile -s cumtime app.py`; `node --cpu-prof app.js`; Go `pprof`; `cargo flamegraph`; the browser's performance panel or Lighthouse for front ends.
- For databases, read the plan with `EXPLAIN`. `EXPLAIN ANALYZE` executes the statement, so for writes run it inside a transaction you roll back, and never against production without approval.
- If a known-good version exists, bisect with a script that fails when the timing passes a threshold well above normal noise.
