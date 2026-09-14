---
name: pr-review-audit
description: Reviews a pull request diff for security, correctness, and architectural risk before merge. Checks authentication and authorization boundaries on new routes, unhandled promise rejections and swallowed errors, N+1 queries and unindexed lookups, breaking API or schema changes, and regression test coverage for each behavioural change. Use when reviewing a PR, auditing a branch diff, or deciding whether a change is safe to merge. Do not use for style, formatting, or lint-level feedback, and do not use it to review code outside the diff.
---

# Pull Request Review Audit

Reviews a diff the way a senior engineer does on a Friday afternoon: looking for
the things that page someone at 3am, not the things a linter already catches.

## When to use this skill

- Reviewing a pull request before approving or merging it
- Auditing a long-lived feature branch before it goes to staging
- Checking whether a hotfix introduced a regression risk under time pressure

## When not to use it

- Formatting, naming, or import-order feedback — your linter owns that
- Full-repository architecture reviews — this skill is scoped to a diff
- Greenfield code with no diff to compare against

## Inputs

Ask for these if they are not already in context:

1. The diff: `git diff <base>...HEAD` (three dots — you want the merge base)
2. Any migration or schema files in the changeset
3. The test files touched by the change

## Procedure

Work through these in order. Do not skip ahead to whatever looks most
interesting.

### 1. Authorization boundaries

For every new or modified route, handler, resolver, or exported server action:

- Is an authentication check present, and does it run *before* any data access?
- Is the authorization check scoped to the acting user, or does it only confirm
  that *someone* is logged in? `findById(req.params.id)` without an ownership
  check is the single most common real vulnerability in application PRs.
- Did middleware get bypassed by registering the route on a different router?

### 2. Error handling

- Every `await` is inside a `try` or has a `.catch`, or the caller demonstrably
  handles rejection.
- No `catch {}` that swallows an error without logging or rethrowing.
- Errors returned to clients do not leak stack traces, SQL, or internal paths.

### 3. Data access

- New queries inside a loop — the N+1 pattern. Look for `await` inside `for`,
  `map`, or `forEach`.
- Lookups on columns with no index, especially foreign keys and `WHERE` columns
  added in this PR.
- Queries with no `LIMIT` on a table that grows unboundedly.

### 4. Contract changes

- Renamed or removed response fields, changed types, changed nullability.
- Enum values removed — existing rows may still hold them.
- Any of the above without a version bump or deprecation path.

### 5. Test coverage

- Each bug fix has a test that fails without the fix.
- Each new branch of behaviour has at least one case.
- Tests assert on behaviour, not on implementation details that will churn.

## Output contract

Produce exactly this structure. No preamble.

```
### Verdict
APPROVE | REQUEST_CHANGES | BLOCK_SECURITY

### Blocking
- `path/to/file.ts:42` — what is wrong and what happens in production
  Fix: the concrete change to make

### Non-blocking
- `path/to/file.ts:88` — observation and suggested improvement

### Not reviewed
- Anything in the diff you could not assess, and why
```

If there are no blocking findings, write `None` under that heading rather than
omitting it. Always populate "Not reviewed" honestly — a review that claims
total coverage it does not have is worse than one that names its gaps.

## Worked example

**Diff adds:**

```js
app.get('/api/invoices/:id', async (req, res) => {
  const invoice = await db.invoice.findUnique({ where: { id: req.params.id } })
  res.json(invoice)
})
```

**Expected finding:**

```
### Verdict
BLOCK_SECURITY

### Blocking
- `routes/invoices.ts:12` — no authorization check. Any authenticated user can
  read any invoice by guessing or enumerating an id, including other tenants'
  billing data.
  Fix: scope the query to the caller —
  `where: { id: req.params.id, organizationId: req.user.organizationId }`
- `routes/invoices.ts:13` — a missing invoice returns `null` as HTTP 200.
  Fix: return 404 when the record is absent.
```

## Common mistakes when using this skill

- Reviewing files the diff did not touch. Stay inside the changeset.
- Reporting every finding as blocking. If everything is urgent, nothing is.
- Assuming a missing test means missing coverage — check whether an existing
  test already exercises the path before flagging it.
