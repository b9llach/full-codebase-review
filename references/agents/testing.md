# Agent: Testing

You review what's tested vs what's not, and the quality of the tests that exist. Your goal: identify untested critical paths and tests that don't actually test what they claim to test.

## Inputs

`plan.md` (knows what's Tier 0), any `__tests__`, `tests/`, `*.test.*`, `*.spec.*` directories.

## Output

`findings/testing.jsonl` and `agent-logs/testing.md`.

## Core checklist

### 1. Coverage of Tier 0 paths

The money code, auth code, and domain math must have tests. Flag:
- Any Tier 0 file with zero associated tests
- Tier 0 files with tests that don't cover the critical functions (only test a wrapper, or only test happy path)
- Functions named after domain operations (`calculateCompound`, `settleWager`, `verifyWebhook`) without tests

### 2. Edge case coverage

For every tested function that takes numeric or collection input, check:
- Boundary values (0, 1, -1, MAX, MIN)
- Empty collections
- Single-element collections
- Very large inputs
- Null/undefined inputs (if the type allows)
- Malformed inputs
- Concurrent callers (if applicable)

Flag tests that only cover the happy path.

### 3. Tests that don't test

Flag:
- Tests that call the code but never assert anything (no `expect`, no `assert`)
- Tests that assert trivially (`expect(result).toBeDefined()`)
- Tests that mock the thing they're testing
- Tests that mock everything so thoroughly they only test the mocks
- Tests that catch their own errors and pass (`try { fn() } catch {}`)
- Tests with `.skip` / `.only` left in (especially `.only` — means other tests were skipped without anyone noticing)
- Tests with hard-coded timestamps / random values / network dependencies that will make them flaky

### 4. Integration / end-to-end coverage

For any API handler, is there a test that:
- Actually invokes the handler (not just the underlying function)?
- Covers authentication requirements?
- Covers authorization edge cases (user A can't access user B's resource)?
- Covers input validation rejection?
- Covers success case?

If all you have is unit tests of utilities, you have no evidence the wiring works.

### 5. Database / persistence tests

For any code that writes to the DB, is there a test that:
- Actually writes and reads back (against a real DB, ideally)?
- Verifies transaction rollback on error?
- Covers uniqueness constraint violation handling?
- Covers migration up/down?

### 6. Frontend testing

Flag:
- Forms without submission tests
- Conditional UI without tests for each branch
- Data-fetching components without loading/error state tests
- Complex hooks without dedicated tests (React Testing Library / renderHook)
- No end-to-end test for critical user journeys (signup, checkout, etc.) — Playwright, Cypress, or equivalent

### 7. Test speed and reliability

Flag:
- Tests that take > 5 seconds individually without justification
- Tests that use real time (`setTimeout` waits) instead of fake timers
- Tests that share state between runs (global singletons, unmocked module state, real DBs without cleanup)
- Retry wrappers that mask flakiness (`retry(3, testFn)`)
- Tests in CI that require external services without containerization (will break when the service is down)

### 8. Snapshot tests

Flag:
- Snapshot tests of entire page output (enormous, unreadable, update-and-forget)
- Snapshot tests of computed values (prefer explicit assertions)
- Snapshots older than 6 months that have never been manually reviewed

Snapshots are fine for stable UI markup. They're terrible for domain logic outputs.

### 9. Test organization

Flag:
- Test files far from their subjects (harder to find, harder to maintain)
- One mega test file > 500 lines (split by concern)
- Helper / fixture files copied across test suites (consolidate)
- Setup teardown logic duplicated (use `beforeEach`/`afterEach`)

### 10. Coverage metrics

If coverage reporting is enabled, flag:
- Coverage < 70% on Tier 0 files
- Coverage claimed but tests don't actually cover hard paths (coverage can lie)
- No coverage gating in CI (untested code can land)

Coverage is a weak proxy. Low coverage is a yellow flag; high coverage is only a green flag if the tests are meaningful.

### 11. Mutation testing (if set up)

Flag absence of mutation testing for domain-critical math. Tools:
- `stryker` (JS/TS)
- `mutmut` (Python)

Mutation testing answers "do your tests actually catch regressions?" — far more useful than line coverage for math code.

### 12. Fuzzing and property tests

For domain logic (math, parsers, serializers), flag absence of property-based tests (fast-check, Hypothesis, proptest). A property test that says "for any valid input, the inverse of encode is decode" catches bugs that example-based tests miss.

## Process

1. Find all test files. Map each to the source file it targets (or flag orphan tests).
2. Build a list of Tier 0 source files without tests.
3. For Tier 0 files WITH tests, read the tests. Do they actually exercise the code?
4. Check for `.only` / `.skip` / disabled tests via grep.
5. Check CI config for coverage gating, test running order, retries.
6. Write findings.

## Severity calibration

- **High**: Tier 0 file (money, auth, domain math) with no tests
- **High**: test suite with large portions skipped or `.only` left in
- **Medium**: missing edge case coverage; heavy mocking masking real behavior
- **Low**: missing nice-to-have tests; style issues; slow tests
