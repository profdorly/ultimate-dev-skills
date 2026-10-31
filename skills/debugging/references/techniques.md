# Debugging techniques

## Reading stack traces

Find the frame where control leaves the project's own code for the last time before the error. That frame, or its caller, holds the bug far more often than a library frame does.

| Language | How to read it | Useful switches |
|---|---|---|
| Python | The last line is the exception; the frame just above it raised it. With chained exceptions ("During handling of the above exception..." or "The above exception was the direct cause..."), the first traceback printed is the original failure. | `python -X faulthandler` or `PYTHONFAULTHANDLER=1` prints a traceback on segfaults; 3.11+ prints exception groups as nested `+-` blocks. |
| JavaScript / TypeScript (Node) | The top frame threw. `at async` frames mark awaited calls. Compiled TypeScript shows `.js` positions unless source maps are on. | `node --enable-source-maps`; `node --stack-trace-limit=50` (the default keeps 10 frames); `--trace-uncaught`, `--trace-warnings`. Unhandled promise rejections crash the process by default. |
| Java / Kotlin | The top frame threw. The root cause is the last `Caused by:` block. `... N more` means those frames repeat the enclosing trace. | `-XX:+ShowCodeDetailsInExceptionMessages` (default since JDK 15) names the null variable in a `NullPointerException`. |
| Go | `panic:` message, then the panicking goroutine's stack, innermost call first. Each frame is a function line followed by a `file:line` line. | `GOTRACEBACK=all` prints every goroutine; `GOTRACEBACK=crash` also dumps core where supported. |
| Rust | A panic prints the message and location; set `RUST_BACKTRACE=1` (or `full`) for the stack. | For `anyhow`/`eyre` errors, print with `{:?}` to see the whole context chain. |
| C# / .NET | Inner exceptions appear after `--->`; read the innermost first. | `dotnet test --logger "console;verbosity=detailed"` |
| Ruby | The first line is where it raised; the frames below are its callers. | `bundle exec rspec --backtrace` shows frames RSpec filters out. |
| C / C++ | Usually no trace at all; run it under a debugger in batch mode (below) or build with `-fsanitize=address,undefined -g`. | `ulimit -c unlimited` before running to get a core dump. |

## Instrumentation

- **Turn on existing logging first.** Many tools already log what you need: `DEBUG=*` (the `debug` package), `NODE_DEBUG=http,net`, `logging.basicConfig(level=logging.DEBUG)` in Python, `RUST_LOG=debug`, framework debug settings, SQL query logging (SQLAlchemy `echo=True`, the `django.db.backends` logger, Prisma `log: ['query']`, ActiveRecord's development log), `curl -v`, and `GIT_TRACE=1`.
- **Add marked, temporary prints.** Tag each line so you can find and remove all of them, and write to stderr so you don't corrupt stdout output:

  ```python
  print(f"DEBUG-pagination page={page!r} size={size!r} offset={offset!r}", file=sys.stderr)
  ```
  ```js
  console.error('DEBUG-pagination', { page, size, offset });
  ```
  ```go
  fmt.Fprintf(os.Stderr, "DEBUG-pagination page=%v size=%v offset=%v\n", page, size, offset)
  ```
  ```rust
  eprintln!("DEBUG-pagination page={page:?} size={size:?} offset={offset:?}");
  ```

- **Print representations, not strings.** `repr()`, `{:?}`, `%#v`, and `util.inspect` show types, quotes, and whitespace, so you can tell `None` from `"None"`, `1` from `"1"`, and spot a trailing `\r`.
- **Log at the boundaries:** the inputs and outputs of the suspect function, before and after a transformation, and both sides of a branch.
- **Use assertions as probes.** `assert offset >= 0, offset` fails at the first point the state goes wrong, which is often earlier than the visible symptom.
- **Clean up.** `git grep -n 'DEBUG-' -- .` must print nothing before you report.

## Debuggers without an interactive terminal

Interactive debuggers (`pdb`, `node inspect`, `dlv`, the `gdb` prompt) wait for keyboard input and hang an agent. Use batch modes and process dumps instead:

| Need | Command |
|---|---|
| Backtrace of a crashing native program | `gdb -batch -ex run -ex bt --args ./prog arg1` or `lldb --batch -o run -k 'thread backtrace all' -- ./prog arg1` |
| Stacks of a running, stuck native process | `gdb -p <pid> -batch -ex 'thread apply all bt'` or `lldb -p <pid> --batch -o 'thread backtrace all'` |
| Stacks of a stuck Python process | `py-spy dump --pid <pid>` (needs root on macOS, and usually on Linux to attach to a running process; if `sudo` would prompt, ask the user to run it) |
| Python test that hangs | `pytest -o faulthandler_timeout=60 path::test` dumps every thread's stack after 60 seconds |
| Stacks of a stuck Go process | `kill -QUIT <pid>` prints every goroutine and exits the process |
| Stacks of a stuck JVM | `jcmd <pid> Thread.print` or `jstack <pid>` |
| Node tests that never exit | `npx jest --detectOpenHandles`, `npx vitest run --reporter=hanging-process` |

`kill -QUIT` ends a Go process; only send it to a process you started or the user approved.

## Bisecting history

When a commit is known to be good and a later one bad, `git bisect run` finds the first bad commit in about log2(N) steps. Bisect checks out old commits, which can overwrite ignored files such as `.env`, so run it in a throwaway worktree and leave the user's tree alone. Tell the user before you start; the worktree is created outside the repository.

1. Write a test script that exits 0 when the commit is good, 1 when it's bad, and 125 when the commit can't be tested (doesn't build, setup failed). Any exit code above 127, such as a crash from a signal, aborts the whole bisect, so map those to 1 or 125 yourself.
2. Keep the script, and the regression test if it's new, **outside** the worktree: old commits don't contain your new test file.
3. Confirm the script fails on the bad commit for the expected reason and passes on the good one before you trust any result. A missing dependency or config file otherwise marks every commit bad and produces a convincing but wrong answer.

```bash
bisect_tmp=$(mktemp -d "${TMPDIR:-/tmp}/bisect.XXXXXX") || exit 1
printf 'bisect temp: %s\n' "$bisect_tmp"   # record it; shell variables may not survive between tool calls
# Write the test script to "$bisect_tmp/check.sh" and make it executable.
git worktree add --detach "$bisect_tmp/wt" <BAD> || exit 1
(
  cd "$bisect_tmp/wt" || exit 1
  git bisect start <BAD> <GOOD> || exit 1
  git bisect run "$bisect_tmp/check.sh"
  run_rc=$?
  git --no-pager bisect log | tail -n 5
  git bisect reset
  exit "$run_rc"
)
bisect_rc=$?
git worktree remove --force "$bisect_tmp/wt"
rm -rf "$bisect_tmp"   # only this invocation's temp directory
printf 'bisect exit: %s\n' "$bisect_rc"
```

The "is the first bad commit" line in the output names the culprit. Read its diff (`git --no-pager show <sha>`) and confirm the cause in the code; the commit that exposed a bug isn't always the one that contains it. If the git-workflow skill is installed, its recovery reference has a stricter version of this recipe.

Read-only history searches that need no worktree:

- `git --no-pager log -S'<string>' --oneline -- <path>` lists commits that added or removed a string.
- `git --no-pager log -G'<regex>' --oneline -- <path>` lists commits whose diff matches a pattern.
- `git --no-pager log -L <start>,<end>:<path>` shows the history of a line range.
- `git --no-pager blame -L <start>,<end> -- <path>` shows the last commit to touch each line.

## Shrinking the input

When a large input triggers the bug, halve it repeatedly and keep whichever half still fails:

- Lines of a file: `head -n 5000 big.csv > half.csv`, then `sed -n '5001,10000p' big.csv`, and so on (keep the header line for CSVs).
- JSON arrays: `jq '.[0:500]' input.json > half.json`.
- Requests or steps: delete half of the steps in the reproduction script.
- Code: comment out half the calls in a minimal script that still reproduces the failure.

Stop when removing any remaining piece makes the failure disappear. Property-based testing tools (Hypothesis, fast-check, proptest) shrink failing inputs for you; `cvise` and `creduce` shrink C and C++ sources.

## Comparing a good run with a bad one

- Outputs: `diff <(good-command 2>&1) <(bad-command 2>&1)`, or save both to files and `diff -u`.
- Dependency versions: the lockfile diff between the two states is usually enough. Otherwise save `npm ls --all`, `pip freeze`, `go version -m <binary>`, or `cargo tree` to a file on each side and `diff -u` the files; their full output is long.
- Environment, without printing secrets: `env | grep -viE 'token|secret|key|pass|auth|cookie|session' | sort > env-<name>.txt` on both sides, then `diff -u`.
- Config: diff the resolved configuration if the tool can print it (for example `docker compose config`, `npx tsc --showConfig`, `git config --list --show-origin`).
