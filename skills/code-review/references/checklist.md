# Review checklist

Go through the areas where the change carries risk. Each question should lead to either "checked, fine" or a finding with evidence.

## Correctness

- Does the code do what the description and the issue say, on the normal path? Did you run it or trace it?
- Are conditions right: `<` versus `<=`, `and` versus `or`, negations, early returns that skip required work?
- Are units, time zones, currencies, and encodings consistent across the boundary?
- Is every new branch reachable, and does every switch or match handle all cases (including a default)?
- Does a change to shared code (a utility, a base class, a config default) break any other caller?

## Edge cases

- Empty, zero, one, many, the maximum; `null`/`None`/`undefined`/missing keys; empty strings versus missing values.
- Unicode, very long input, whitespace, mixed case, leading zeros.
- Duplicate requests, retries, and the same event delivered twice: is the operation idempotent where it needs to be?
- Pagination and limits: the first page, the last page, a page past the end, a changing data set.
- Dates: month ends, leap days, daylight-saving changes, time-zone boundaries.

## Errors and failure

- Are errors handled where something can be done about them, and propagated otherwise? No swallowed exceptions, no empty `catch`, no ignored error returns.
- Do error messages carry enough context to debug, without leaking secrets or internals to users?
- On partial failure, is state left consistent: transactions, cleanup in `finally`/`defer`, compensating actions?
- Are timeouts set on network calls? Are retries bounded, with backoff, and only for operations that are safe to repeat?
- Are resources released: files, connections, locks, subscriptions, temp files?

## Concurrency

- Is shared mutable state protected? Check-then-act sequences (read, decide, write) without a lock or a database constraint are races.
- Can two requests create the same record? Is there a unique constraint, or only an application-level check?
- Async: is every promise or task awaited? Any blocking call on an event loop?
- Lock ordering, and locks held across I/O.

## Security

Check these whenever the change touches input, output, auth, or secrets. The secure-coding skill, if installed, has the detailed rules.

- Is every new endpoint or handler authenticated, and does it check that the caller may act on this specific object?
- Is input validated at the boundary? Any SQL, shell, HTML, path, URL, or template built from input by string concatenation?
- Are secrets kept out of code, logs, URLs, error messages, and client bundles?
- Was any security control disabled or weakened (TLS verification, CSRF protection, CORS, auth middleware)?
- Do new dependencies come from a verified source? The dependencies skill covers vetting.

## Data and migrations

- Is the migration reversible, or is the irreversibility called out?
- Will it lock a large table or rewrite it? Is old code still running against the new schema during the deploy, and new code against the old one?
- Are backfills batched and idempotent? Are data changes separate from schema changes?
- Do new columns have sensible defaults and constraints? Are indexes added for new query patterns?
- The database-migrations skill, if installed, has the rules for safe schema changes.

## API and compatibility

- Does the change remove or rename anything public: endpoints, fields, CLI flags, environment variables, config keys, events, exported functions? Is it versioned or announced as breaking?
- Are new fields optional for old clients? Are defaults unchanged for existing callers?
- Do serialized formats (JSON, stored files, cache entries, queue messages) still read data written by the previous version?
- Is the semantic version bump right for what consumers can observe?

## Performance

- Queries inside loops (N+1), missing indexes for new filters or sorts, unbounded `SELECT`s without a limit.
- Work repeated per request that could be done once; large objects loaded into memory where streaming would do.
- Hot paths: allocations in tight loops, regular expressions compiled on every call, synchronous I/O.
- Ask for measurements when a change claims to improve performance.

## Tests

- Is there a test for each new behavior and for the bug being fixed? Would it fail if the change were reverted?
- Do the tests assert outcomes, or just that nothing threw?
- Are they deterministic (time, randomness, network, order)?
- Were any tests deleted, skipped, or loosened? Is each such change justified?
- Were snapshots updated? Does every snapshot change match the intent?

## Observability and operations

- Are important failures logged or reported with enough context, at the right level, without personal data?
- Are new configuration values documented, with safe defaults, and present in every environment that needs them?
- Can the change be rolled back safely? Is it behind a feature flag if it's risky?

## Readability and design

- Do names say what things are and do? Would a newcomer understand the code without the PR description?
- Is there duplicated logic that already exists elsewhere in the codebase?
- Is the abstraction proportional to the problem, neither copy-paste nor a framework for one use?
- Do comments explain why, and are they still true after this change?

## Dependencies

- Is each new dependency necessary, maintained, licensed compatibly, and verified to be the real package?
- Is the lockfile updated consistently with the manifest, and only as much as the change requires?

## Language-specific traps

| Language | Look for |
|---|---|
| JavaScript / TypeScript | `==` instead of `===`; missing `await`; `async` callbacks in `forEach`; mutating props or state in place; `any` and non-null assertions (`!`) hiding real nulls; numbers compared as strings; `parseInt` without a radix on user input; floating promises in event handlers |
| React | Missing or wrong dependencies in `useEffect`/`useMemo`/`useCallback`; state updated from stale closures; array index as `key` for reorderable lists; effects without cleanup; `dangerouslySetInnerHTML` |
| Python | Mutable default arguments; late-binding closures in loops; bare `except:` or `except Exception: pass`; `is` for value comparison; modifying a list while iterating it; naive versus aware `datetime`; blocking calls inside `async def` |
| Go | Ignored `err`; `defer` inside loops; unclosed `resp.Body`; goroutines without a way to stop; a nil pointer stored in an interface (not `== nil`); maps written concurrently; `context` not passed through |
| Rust | `unwrap()`/`expect()` in library or request-handling code; blocking calls in async code; `clone()` hiding ownership problems; integer overflow in release builds |
| Java / Kotlin | `equals` without `hashCode`; resources not in try-with-resources; catching `Exception` broadly; mutable static state; `Optional.get()` without a check; Kotlin `!!` |
| SQL | String-built queries; `SELECT *` in application code; missing `WHERE` in `UPDATE`/`DELETE`; N+1 patterns; transactions held across network calls; implicit type casts that prevent index use |
| Shell | Unquoted variables; missing `set -u` or `${var:?}` before `rm -rf "$var/..."`; parsing `ls`; GNU-only flags in scripts that also run on macOS (the shell-scripts skill covers these) |
