---
name: git-commit-convention
description: Writes Conventional Commits messages from a staged diff and determines the resulting semantic version bump. Selects the correct type and scope, writes an imperative subject under 72 characters, adds a body explaining why the change was made, and marks breaking changes with a BREAKING CHANGE footer. Use when committing staged changes, rewriting a vague commit message, or checking whether a change is a patch, minor, or major release. Do not use for squash-merge release notes or changelog generation across many commits.
---

# Conventional Commit Writer

Turns a staged diff into a commit message that a release tool can parse and a
human can still read six months later.

## When to use this skill

- Writing a commit message for staged changes
- Rewriting a message like "fix stuff" into something meaningful
- Deciding whether a change is a patch, minor, or major bump

## When not to use it

- Generating a changelog from many commits — different job
- Writing PR descriptions, which summarise a branch rather than one change

## Inputs

- `git diff --staged` (the staged diff, not the working tree)
- The current version from `package.json`, `Cargo.toml`, or equivalent, if a
  version bump needs to be reported

## Format

```
<type>(<optional scope>): <subject>

<body>

<footers>
```

### Types and their version impact

| Type | Meaning | Bump |
|---|---|---|
| `feat` | A new capability visible to users | minor |
| `fix` | A bug fix visible to users | patch |
| `perf` | Faster or lighter with no behaviour change | patch |
| `refactor` | Restructuring with no behaviour change | none |
| `docs` | Documentation only | none |
| `test` | Tests only | none |
| `build` | Build system, bundler, or dependencies | none |
| `ci` | Pipeline configuration | none |
| `chore` | Maintenance with no src change | none |

Any type with a `BREAKING CHANGE:` footer is a **major** bump, including `fix`.

## Procedure

1. **Read the diff before naming the type.** The type describes what the change
   does for a consumer, not which directory it touched. A change under `src/`
   that only renames internals is `refactor`, not `feat`.
2. **Pick one type.** If the diff genuinely does two things, say so and
   recommend splitting the commit rather than inventing a compound type.
3. **Scope** is the package, module, or surface — `auth`, `api`, `parser`.
   Omit it rather than inventing a vague one like `core` or `misc`.
4. **Subject**: imperative mood, lower case, no trailing period, under 72
   characters. "add retry to webhook sender", not "Added retries." or
   "This commit adds retries".
5. **Body**: explain *why*, not *what* — the diff already shows what. Wrap at
   72 columns. Skip the body only when the subject is genuinely complete.
6. **Footers**: `BREAKING CHANGE: <description>` for incompatible changes,
   `Refs: #123` or `Closes: #123` for issue links.

## Output contract

Return the commit message in a fenced block, followed by one line:

```
Version impact: patch | minor | major | none (current X.Y.Z -> A.B.C)
```

Nothing else. No commentary around the message.

## Weak versus strong

**Weak:**

```
fix: bug fixes
```

Unparseable intent, no scope, tells a future reader nothing.

**Strong:**

```
fix(webhooks): retry delivery on 5xx with exponential backoff

Stripe returns 502 during their deploys, which was silently dropping
payment confirmations. Deliveries now retry five times over roughly
ten minutes before landing in the dead-letter queue.

Refs: #412
```

**Breaking:**

```
feat(api): return ISO 8601 timestamps in all responses

Timestamps were Unix epoch integers, which forced every client to know
which fields were dates. All timestamp fields are now ISO 8601 strings.

BREAKING CHANGE: `createdAt` and `updatedAt` are ISO 8601 strings
rather than integers. Clients parsing them as numbers must be updated.
```

## Common mistakes

- Using `feat` for internal refactors because the diff is large. Size is not
  the signal; consumer-visible behaviour is.
- Forgetting that a `fix` with a `BREAKING CHANGE` footer is a major bump.
- Writing the body as a restatement of the diff instead of the reason for it.
- Scoping by file path (`fix(src): ...`) rather than by module.
