---
name: test-coverage-audit
description: Finds behaviour in a diff that no test exercises, and writes the missing cases. Maps each new or changed branch, error path, boundary condition, and state transition to an existing test or flags it as uncovered, then drafts the specific tests needed. Use when reviewing test coverage on a pull request, before merging a bug fix, or when a coverage percentage looks healthy but bugs still ship. Do not use for raising a coverage number for its own sake, for generating tests against untouched legacy code, or for performance benchmarking.
---

# Test Coverage Audit

Line coverage tells you which lines ran. It does not tell you whether anything
was actually verified. This skill audits *behaviour* coverage on a diff and
writes the cases that are missing.

## When to use this skill

- Reviewing whether a PR is adequately tested before approving it
- After a bug fix, to confirm there is a test that fails without the fix
- When coverage reports look fine but regressions keep reaching production

## When not to use it

- Chasing a coverage percentage target — that produces assertions like
  `expect(result).toBeDefined()` and teaches you nothing
- Retrofitting tests onto legacy code untouched by the current change
- Load or performance testing

## Inputs

1. `git diff <base>...HEAD`
2. The existing test files for the changed modules
3. How to run the suite (`npm test`, `pytest`, `cargo test`)

## Procedure

### 1. Enumerate the behaviour, not the lines

For each changed function, list every distinct outcome it can produce:

- Each branch of every conditional
- Each early return and guard clause
- Each thrown or returned error
- Each boundary: empty, one, many, null, zero, negative, maximum
- Each state transition, if the code is a state machine or reducer

### 2. Map each behaviour to a test

For each item, find the test that asserts it. Record one of:

- `covered` — a test exists and asserts the *outcome*, not merely that the
  function ran
- `executed but unasserted` — a test runs this path but never checks the
  result. This is the case coverage tools report as green and is the most
  dangerous category
- `uncovered` — no test reaches this path

### 3. Judge what matters

Not every uncovered path deserves a test. Prioritise:

- **High**: error paths, auth decisions, money, data loss, anything in the bug
  this PR fixes
- **Medium**: boundaries and state transitions
- **Low**: logging, trivial getters, defensive branches that cannot be reached

### 4. Write the missing high and medium cases

Match the file's existing style, framework, and naming. Each test must fail if
the behaviour regresses — verify this by reasoning about what breaking the code
would do to the assertion.

## Output contract

```
### Coverage summary
<n> behaviours identified — <n> covered, <n> executed but unasserted, <n> uncovered

### Gaps by priority
[HIGH] `file.ts:fn` — <behaviour> is <status>
       Why it matters: <concrete failure this allows>

### Missing tests
<runnable test code, in the project's existing style>

### Deliberately skipped
- <path> — <why a test here would not earn its maintenance cost>
```

Always populate "Deliberately skipped". A review that demands tests for
everything gets ignored wholesale.

## Worked example

**Diff:**

```js
function applyDiscount(total, code) {
  if (!code) return total
  const pct = CODES[code]
  if (!pct) throw new InvalidCodeError(code)
  return total - (total * pct)
}
```

**Existing test:** one case, a valid code.

**Expected output:**

```
### Coverage summary
5 behaviours identified — 1 covered, 0 executed but unasserted, 4 uncovered

### Gaps by priority
[HIGH] `pricing.js:applyDiscount` — unknown code throws InvalidCodeError: uncovered
       Why it matters: a typo'd code currently 500s the checkout instead of
       being rejected cleanly. Nothing verifies the error type.
[MEDIUM] `pricing.js:applyDiscount` — falsy code returns total unchanged: uncovered
[MEDIUM] `pricing.js:applyDiscount` — total of 0: uncovered
[LOW] `pricing.js:applyDiscount` — 100% discount returns 0: uncovered

### Missing tests
test('throws InvalidCodeError for an unknown code', () => {
  expect(() => applyDiscount(100, 'NOPE')).toThrow(InvalidCodeError)
})

test('returns the total unchanged when no code is given', () => {
  expect(applyDiscount(100, null)).toBe(100)
  expect(applyDiscount(100, '')).toBe(100)
})

test('handles a zero total', () => {
  expect(applyDiscount(0, 'SAVE10')).toBe(0)
})

### Deliberately skipped
- The 100% discount case — covered by the same arithmetic as the valid-code
  test; a separate case adds maintenance without adding signal.
```

## Common mistakes

- Counting a path as covered because a test *executes* it. Ask what the test
  would catch if that line were deleted.
- Writing tests that assert implementation detail — internal call counts, private
  state — which break on every refactor and get deleted in frustration.
- Flagging every uncovered branch as high priority, which makes the report
  unusable.
