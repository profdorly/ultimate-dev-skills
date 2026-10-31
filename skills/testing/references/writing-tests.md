# Writing deterministic tests

Use whatever the project already uses. The tools below are common defaults; adding one is a new dependency, so ask first.

## Controlling what tests depend on

| Concern | Python | JavaScript / TypeScript | Go | Rust | Java / Kotlin | Ruby |
|---|---|---|---|---|---|---|
| Current time | `time-machine` or `freezegun`; or pass a clock | `vi.useFakeTimers()` + `vi.setSystemTime()`, `jest.useFakeTimers()`; or pass a clock | Pass a `func() time.Time` or a clock interface; `testing/synctest` (Go 1.25+) fakes time inside a test bubble | Pass a clock; `tokio::time::pause()` for Tokio | Inject `java.time.Clock`; `Clock.fixed(...)` in tests | `travel_to` (Rails), `timecop` |
| Timers and timeouts | `asyncio` with a fake clock, or short injected durations | Fake timers + `vi.advanceTimersByTime()` / `jest.advanceTimersByTime()` | `testing/synctest`, or injected durations | `tokio::time::advance()` | Awaitility for polling instead of sleeps | |
| Randomness | Seed it: `random.seed(1)`, `Faker.seed(1)`; pass a `random.Random` instance | Inject the generator; `faker.seed(1)` | `rand.New(rand.NewPCG(1, 2))` (`math/rand/v2`) | `StdRng::seed_from_u64(1)` | `new Random(1)` | `srand(1)`, `Faker::Config.random = Random.new(1)` |
| HTTP calls | `responses` (requests), `respx` (httpx), `pytest-httpserver` | `msw`, `nock`, undici `MockAgent` | `httptest.NewServer` | `wiremock`, `mockito` | WireMock, OkHttp `MockWebServer` | `webmock` |
| Files | `tmp_path` fixture | `fs.mkdtemp(path.join(os.tmpdir(), 'x-'))` | `t.TempDir()` | `tempfile` crate | JUnit 5 `@TempDir` | `Dir.mktmpdir` |
| Environment variables | `monkeypatch.setenv` | `vi.stubEnv`, or save and restore `process.env` | `t.Setenv` | Pass config explicitly (setting env vars is `unsafe` in Rust 2024 and races between threads) | System Stubs, or inject config | `climate_control` |
| Generated inputs | Hypothesis | fast-check | `testing/quick`, `pgregory.net/rapid` | proptest, quickcheck | jqwik | `rantly` |

When a dependency is hard to control (a global clock, a module-level client), the fix is usually to pass it in as a parameter or constructor argument. That is a small, legitimate design change, unlike a test-only branch in production code.

## Mocks, stubs, and fakes

- **Stub** a return value when the test only needs the collaborator to answer.
- **Mock** (assert on calls) only when the call itself is the behavior: an email was sent, a payment was charged, an event was published.
- **Fake** (a working in-memory implementation) for collaborators with real logic, such as a repository or a queue. Fakes catch bugs that call-by-call mocks miss.
- Mock at the edge of the system (HTTP, the clock, the message broker), not the module under test or its simple helpers.
- Keep mocks honest: a mock that accepts any arguments, or returns data the real service never returns, makes the test pass against code that would fail in production. Prefer recorded real responses, and typed mocks (`spec_set=` / `autospec=True` in Python's `unittest.mock`, typed fakes in TypeScript) so a changed interface breaks the test.
- Patch where the name is looked up, not where it's defined: in Python, `patch("myapp.billing.send_email")` if `billing.py` did `from myapp.mail import send_email`.

## Databases in tests

- Use the project's established approach: a transaction rolled back after each test (Django `TestCase`, Rails transactional tests), a fresh schema per run, or containers (testcontainers, `docker compose`).
- SQLite in memory is not Postgres or MySQL. Types, constraints, locking, JSON operators, and case sensitivity differ. If production runs Postgres, behavior that depends on the database needs a test against Postgres.
- Give each test its own data, created by factories inside the test, never rows left behind by another test.
- Never point tests at a database whose URL you haven't checked is local and disposable.

## Async code

- Await everything the test starts. A promise or task created without `await` can fail after the test has already passed, or leak into the next test.
- In JavaScript, return or await the promise; `forEach` with an async callback doesn't wait. Use `for ... of` with `await`, or `await Promise.all(...)`.
- In Python, use the project's plugin (`pytest-asyncio`, `anyio`) and its marker or mode.
- To assert on something eventually true, use a waiting assertion with a deadline (`await expect(locator).toBeVisible()`, `vi.waitFor`, Awaitility, `require.Eventually` in testify), never a fixed sleep.

## Testing legacy or untested code

Before changing code that has no tests, pin its current behavior with **characterization tests**:

1. Call the code with representative inputs, including odd ones.
2. Record what it returns today as the expected value, even if the value looks wrong.
3. Mark suspicious results with a comment (`# current behavior; looks wrong, see #123`) and tell the user about them; don't silently "fix" them in the same change.
4. Refactor with those tests green, then change behavior in a separate step with its own tests.

## Anti-patterns

| Pattern | Problem | Instead |
|---|---|---|
| A test without assertions, or one that only checks "doesn't throw" | Passes no matter what the code returns | Assert on the result |
| `if` or loops in the test body that decide what to assert | The test can quietly assert nothing | Separate tests, or a parameterized table |
| Expected value computed by the same logic as the code | Repeats the bug in the test | Write the expected value literally |
| A shared mutable fixture changed by tests | Order-dependent results | Fresh data per test |
| Snapshotting a whole page or large object | Every unrelated change breaks it, so people update it without reading | Assert on the fields that matter; keep snapshots small |
| Testing private methods | Breaks on refactor | Test through the public interface |
| `sleep(2)` before an assertion | Slow, and still flaky | Wait for the condition |
| Catching the exception in the test and passing | Hides the failure | `pytest.raises`, `expect(...).toThrow`, `assertThrows` |
