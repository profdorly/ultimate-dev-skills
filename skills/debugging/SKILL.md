---
name: debugging
description: Use when fixing a bug, error, exception, crash, stack trace, wrong output, regression, hang, memory leak, or broken build; when something fails only in CI, in production, or on one machine; or after a fix attempt failed. Reproduce first, test one hypothesis at a time, fix the root cause instead of the symptom, and prove the fix with a regression test that fails without it. For flaky tests with no known bug, use testing.
license: MIT
metadata:
  author: profdorly
  version: "0.1.0"
---

# Debugging

A bug is a gap between what you believe the code does and what it actually does. Close that gap with evidence. Guessing produces patches that hide the failure, break something nearby, or come back next week.

## Rules (non-negotiable)

1. **Reproduce before you fix.** Run the failing command yourself and watch it fail for the reported reason. If you can't reproduce it, say so, and never present a fix as verified. See "When you can't reproduce it" in [references/hard-cases.md](references/hard-cases.md).
2. **Read the actual error.** Read the whole message and the whole stack trace, and look for the first error in a log, not the last. Open the file and line it names. Never paraphrase an error you haven't read.
3. **One hypothesis, one change.** Before each check, write down what you expect to see if the hypothesis is true. Revert every experiment that didn't help before you try the next one; stacked speculative edits make the cause impossible to pin down.
4. **Fix the cause, not the symptom.** None of these is a fix: catching and ignoring the exception, `|| true`, a `sleep` or a retry that makes a race pass, special-casing the failing input, loosening a type to `any`, silencing a lint or type error, raising a timeout without knowing why the operation is slow. If one of them really is the right call (a retry around a third-party API that fails transiently, for example), say why and get the user's agreement.
5. **Never edit a test to agree with the bug.** If the test looks wrong, show the evidence (spec, docs, other callers, the commit that introduced it) and ask before changing it. Don't delete, skip, or weaken assertions to get a green run.
6. **Instrumentation is temporary.** Tag every debug print or log line you add with a marker such as `DEBUG-<topic>`, and remove all of them before you report. Grep for the marker to prove they're gone.
7. **Stop after three failed fixes.** Three misses mean your model of the system is wrong. Stop editing, list what you have ruled out and what you are still assuming, and report to the user before a fourth attempt.
8. **Debug locally.** Don't point debugging at production systems, shared databases, or real customer data, and don't run commands with side effects there, without the user's approval. Never paste secrets or personal data from logs into the conversation; name the field instead.
9. **Leave the user's work alone.** Debugging doesn't license reverting, stashing, or resetting changes you didn't make. If the git-workflow skill is installed, its safety rules apply to every git command you run here.

## 1. Pin down the symptom

Write this down before you touch code:

- the exact command, request, or steps that fail, and where (local, CI, which machine or environment);
- the exact output or error, copied rather than summarized;
- expected versus actual behavior, and how you know what's expected (spec, test, docs, the user);
- when it started, if known: a deploy, a dependency bump, a config change, a date.

If the report is vague ("login is broken"), find the missing piece in logs and recent history or ask for it. Don't start fixing a guess.

## 2. Reproduce it

- Run it yourself, using the narrowest command that shows the failure: one test (`pytest path/test_x.py::test_name -x`, `npx vitest run path -t 'name'`, `go test ./pkg -run '^TestName$' -count=1`), one request (`curl -sS -i ...`), or one script. Use the project's wrapper (`uv run`, `bundle exec`, `npm run`) so versions and environment match.
- Make it **fast** (seconds) and **deterministic**: pin random seeds, freeze time, and cut network calls where you can. A reproduction you can run 50 times is worth more than an hour of reading.
- **Shrink it.** Remove inputs, steps, and code until removing anything else makes the failure go away. The minimal case usually points straight at the cause.
- Turn the reproduction into a failing automated test as early as you can, and watch it fail for the expected reason (an assertion about the bug, not an import error). That test becomes your regression test.
- Wrap anything that might hang in a timeout (see [references/hard-cases.md](references/hard-cases.md#hangs-and-deadlocks)).

If it doesn't reproduce, compare environments before you read code: [references/hard-cases.md](references/hard-cases.md).

## 3. Localize it

Find the first point where expected and actual diverge. Work from evidence toward the cause:

- **Read the stack trace.** Find the deepest frame in the project's own code; library frames rarely hold the bug. How to read traces in each language is in [references/techniques.md](references/techniques.md#reading-stack-traces).
- **Check what changed.** If it used to work, the cause is usually in the difference: `git --no-pager log --oneline -20 -- <paths>`, `git --no-pager diff <last-good>..HEAD -- <paths>`, the lockfile diff for dependency changes, and any config or environment change.
- **Bisect when you have a known-good state.** In code, halve the path between the input and the wrong output with assertions or logs. In history, run `git bisect run` in a throwaway worktree ([references/techniques.md](references/techniques.md#bisecting-history)). In data, halve the input until one record remains. In config, toggle half the settings.
- **Inspect state at the divergence point.** Log or assert the values entering and leaving the suspect function, and compare them with what they should be.

## 4. Test one hypothesis at a time

Keep a short log in the conversation so neither you nor the user walks the same dead end twice:

```
Symptom:  GET /tasks?page=2 returns items 11-20; expected 10-19 (0-based offset)
Repro:    pytest tests/api/test_pagination.py::test_second_page -x  (fails 10 of 10 runs)
H1 ✗  stale cache: still fails with the cache disabled
H2 ✓  offset computed as page*size+1: logged offset=11 for page=2, size=10
Cause:    src/api/pagination.py:42 adds 1 to a 0-based offset (introduced in 3f2a91c)
```

For each hypothesis, decide beforehand which result would prove it wrong. A check that can't fail tells you nothing.

## 5. Fix the root cause

- Make the smallest change that removes the cause. Don't bundle refactors or cleanups into the fix.
- Fix it at the right layer. Reject bad input where it enters the system rather than where it finally crashes, and don't paper over a bad value three calls downstream.
- Look for the same mistake elsewhere: search for the pattern you just fixed and list the other occurrences. Fix them only if they're in scope; otherwise report them.
- If the right fix is large, risky, or changes behavior other code relies on, describe the options with their trade-offs and ask before doing it.

## 6. Verify

- The reproduction passes now, and keeps passing over repeated runs if the failure was ever intermittent ([references/hard-cases.md](references/hard-cases.md#intermittent-failures) says how many runs make that credible).
- The regression test fails without the fix and passes with it. If you wrote the test first (step 2), you've already seen the failure. If you wrote the fix first, undo just the fix edit, run the test and watch it fail, then reapply the fix. A test that passes either way guards nothing.
- The surrounding test suite and the project's other checks (lint, types, build) pass.
- Every trace of debugging is gone: `git grep -n 'DEBUG-' -- .` prints nothing, and no temporary files, config changes, or debug flags remain.

## 7. Report

```
Root cause: src/api/pagination.py:42 added 1 to a 0-based offset (since 3f2a91c)
Evidence:   offset logged as 11 for page=2; the regression test failed before the fix
Fix:        compute the offset as (page - 1) * size; validate page >= 1 at the API boundary
Tests:      test_second_page added; pytest tests/api passed (48 tests), mypy clean
Not fixed:  the same +1 pattern in src/export/batch.py:17 (export is out of scope)
Confidence: high; reproduced and verified locally on Python 3.12
```

Report only what you ran and saw. If the fix is unverified (you couldn't reproduce the failure), say so in the first line.

## Stop and ask when

- You can't reproduce the failure and the only way forward is a speculative change.
- Three fix attempts have failed.
- The fix means changing a test's expected value, a public API, a data format, stored data, or a database schema.
- Reproducing needs production access, real customer data, credentials you don't have, or a paid or external service.
- The cause is in a dependency or a system you don't own. Report it with a minimal reproduction and propose a workaround; don't silently patch vendored or installed code.
- You find a security problem while debugging.

## Tempting shortcuts

| Shortcut | Why not |
|---|---|
| "I can see the bug, no need to reproduce it." | Then you can't show your fix worked, and the bug you see may not be the one reported. |
| "Wrap it in a try/except and log it." | The failure still happens, now silently, and the bad state travels further. |
| "A retry or a sleep makes it pass." | A race you've made rarer is still a race. It comes back under load or on a slower machine. |
| "Update the expected value so the test passes." | That makes the test agree with the bug. Find out which one is wrong first. |
| "It's probably the library." | Usually it isn't. Prove it with a reproduction that uses only the library. |
| "While I'm here, I'll refactor this." | Mixed changes hide which edit fixed the bug and make the fix harder to review and revert. |
| "One more try with a small tweak." | After three misses your mental model is wrong. Step back and re-check your assumptions. |
| "It works on my machine." | Then the environment is the variable. Compare environments ([references/hard-cases.md](references/hard-cases.md)). |

## References

| Situation | Read |
|---|---|
| Reading stack traces by language, adding instrumentation, using debuggers without an interactive terminal, bisecting history, shrinking inputs, comparing good and bad runs | [references/techniques.md](references/techniques.md) |
| Can't reproduce, intermittent failures, only in CI, only in production, hangs and deadlocks, memory growth, races, wrong results without an error, performance regressions | [references/hard-cases.md](references/hard-cases.md) |
