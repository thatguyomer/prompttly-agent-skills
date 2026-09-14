---
name: refactor-safety-net
description: Builds characterization tests that pin down what code currently does before you change it. Identifies observable behaviour and side effects, writes tests asserting the existing output including its bugs, and defines which behaviours must be preserved versus deliberately changed. Use before refactoring untested or poorly understood code, extracting a module, or replacing an implementation behind a stable interface. Do not use for writing tests for new features, for code that already has good behavioural coverage, or as a substitute for reading the code.
---

# Refactor Safety Net

Before you change code you do not fully understand, pin down what it currently
does. Characterization tests describe present behaviour — including bugs — so
that a refactor that changes behaviour fails loudly instead of silently.

## When to use this skill

- Refactoring a function or module with little or no test coverage
- Extracting code into a new module or service
- Replacing an implementation behind an interface that must stay stable
- Inheriting code you did not write and need to modify safely

## When not to use it

- Writing tests for new features — those assert intended behaviour, not current
- Code that already has solid behavioural tests
- As a substitute for reading the code. These tests describe *what*, never *why*

## The core distinction

A normal test asserts what the code **should** do. A characterization test
asserts what it **currently** does. When they disagree, the current behaviour
wins — because something downstream may depend on the bug.

Write the bug into the test, with a comment marking it. Then decide separately
whether to fix it.

## Inputs

- The target function, module, or class
- Its callers, so you know which behaviours are actually observed
- How to run the suite

## Procedure

### 1. Map the observable surface

- Return values for representative inputs
- Thrown exceptions: type and message
- Side effects: writes, network calls, mutated arguments, emitted events
- Ordering, if callers could depend on it

Mutated input arguments are the most commonly missed category and the most
common source of refactoring regressions.

### 2. Probe boundaries

Empty, null, undefined, zero, negative, very large, wrong type, duplicate
entries. Record what actually happens — including a thrown `TypeError`. That is
behaviour too.

### 3. Write the tests

- Name them for the observed behaviour: `returns null when the list is empty`
- Assert exact values, not shapes. `toBe(0)` not `toBeDefined()`
- Where current behaviour is wrong, assert it anyway and mark it:

  ```js
  // CHARACTERIZATION: returns 0 rather than throwing for a negative
  // quantity. Likely a bug. Preserved deliberately — see issue #— before
  // changing, confirm no caller relies on it.
  expect(calculateTotal(-1, 10)).toBe(0)
  ```

### 4. Confirm the net actually catches things

For each test, reason about a plausible refactoring mistake and confirm the
test would fail. A safety net nothing can fall through is not a net.

### 5. State the contract

Before refactoring, declare which behaviours are frozen and which you intend to
change. Anything in the second list should have its characterization test
updated *in the same commit* as the behaviour change, never quietly deleted.

## Output contract

```
### Observable surface
- <behaviour> -> <current result>

### Characterization tests
<runnable test code>

### Suspected bugs preserved
- <behaviour> — why it looks wrong, and what to check before changing it

### Refactor contract
Frozen: <behaviours that must not change>
Intended to change: <behaviours this refactor will deliberately alter>
```

## Worked example

**Target:**

```js
function parseTags(input) {
  if (!input) return []
  return input.split(',').map(t => t.trim()).filter(Boolean)
}
```

**Expected output:**

```
### Observable surface
- 'a,b'        -> ['a', 'b']
- 'a, b ,c'    -> ['a', 'b', 'c']
- ''           -> []
- null         -> []
- 'a,,b'       -> ['a', 'b']
- 'a,a'        -> ['a', 'a']    (duplicates preserved)
- 0            -> []            (falsy, not a string)
- ['a']        -> TypeError: input.split is not a function

### Characterization tests
test('splits and trims comma-separated tags', () => {
  expect(parseTags('a, b ,c')).toEqual(['a', 'b', 'c'])
})

test('returns an empty array for falsy input', () => {
  expect(parseTags('')).toEqual([])
  expect(parseTags(null)).toEqual([])
  // CHARACTERIZATION: 0 is falsy, so it returns [] rather than throwing.
  expect(parseTags(0)).toEqual([])
})

test('drops empty segments', () => {
  expect(parseTags('a,,b')).toEqual(['a', 'b'])
})

test('preserves duplicates', () => {
  // CHARACTERIZATION: no de-duplication. Callers may rely on counts.
  expect(parseTags('a,a')).toEqual(['a', 'a'])
})

test('throws on non-string input', () => {
  // CHARACTERIZATION: no type guard. Swallowing this in a refactor would
  // hide caller bugs that currently fail loudly.
  expect(() => parseTags(['a'])).toThrow(TypeError)
})

### Suspected bugs preserved
- Duplicates are not removed. Looks unintended for a tag parser, but a caller
  counting occurrences would break silently if deduplicated. Check call sites.
- Array input throws rather than being handled. Preserving the throw is safer
  than silently coercing.

### Refactor contract
Frozen: falsy handling, trimming, empty-segment removal, duplicate preservation
Intended to change: nothing in this pass
```

## Common mistakes

- Asserting shapes instead of values, which lets a refactor change results while
  the tests stay green.
- Quietly "fixing" a bug while writing the net, so the tests describe neither
  the old nor the new behaviour.
- Forgetting mutated arguments and emitted side effects.
- Deleting a characterization test because it fails after a refactor. That
  failure is the net doing its job — decide deliberately, then update it.
