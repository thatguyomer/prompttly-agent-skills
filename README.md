# Prompttly Agent Skills

Production-ready [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview)
skills for engineering workflows. Each one is a single `SKILL.md` with an
explicit procedure, an output contract, and a worked example — written to be
read and adapted, not just installed.

| Skill | What it does |
|---|---|
| [`pr-review-audit`](skills/pr-review-audit/SKILL.md) | Reviews a PR diff for authorization gaps, swallowed errors, N+1 queries, contract breaks, and missing test coverage |
| [`test-coverage-audit`](skills/test-coverage-audit/SKILL.md) | Finds behaviour in a diff that no test exercises — including paths that run but are never asserted — and writes the missing cases |
| [`db-migration-check`](skills/db-migration-check/SKILL.md) | Audits a migration for table locks, backward-incompatible changes, and unsafe backfills, and rewrites it |
| [`api-contract-review`](skills/api-contract-review/SKILL.md) | Checks a REST or GraphQL change for breaking field, type, nullability, and enum changes, and gives an additive migration path |
| [`refactor-safety-net`](skills/refactor-safety-net/SKILL.md) | Writes characterization tests that pin current behaviour — bugs included — before you change code you don't fully understand |
| [`git-commit-convention`](skills/git-commit-convention/SKILL.md) | Writes a Conventional Commits message from a staged diff and reports the resulting semver bump |
| [`incident-postmortem`](skills/incident-postmortem/SKILL.md) | Drafts a blameless postmortem that separates trigger from root cause and produces owned, verifiable action items |

## Install

Copy the skills into your personal skills directory to make them available in
every project:

```bash
git clone https://github.com/thatguyomer/prompttly-agent-skills.git
cp -r prompttly-agent-skills/skills/* ~/.claude/skills/
```

Or install a single skill:

```bash
cp -r prompttly-agent-skills/skills/pr-review-audit ~/.claude/skills/
```

For a skill scoped to one repository, copy it into that repo instead and commit
it so the whole team picks it up:

```bash
mkdir -p .claude/skills
cp -r prompttly-agent-skills/skills/db-migration-check .claude/skills/
```

Start a new Claude Code session afterwards — skills are indexed at startup.

## Use

Invoke a skill directly:

```
/pr-review-audit review the diff between main and HEAD
```

Or describe the task and let Claude Code match it against the skill's
description:

```bash
claude "check this migration for anything that will lock the orders table"
```

## Structure

```
skills/
  api-contract-review/SKILL.md
  db-migration-check/SKILL.md
  git-commit-convention/SKILL.md
  incident-postmortem/SKILL.md
  pr-review-audit/SKILL.md
  refactor-safety-net/SKILL.md
  test-coverage-audit/SKILL.md
```

Every `SKILL.md` carries YAML frontmatter with a `name` and a `description`
under 1,024 characters. The description is what Claude Code matches against at
runtime, so it states both when to use the skill and when not to — vague
descriptions cause skills to fire on unrelated prompts.

## Writing your own

The pattern these follow:

1. **Frontmatter** — `name` and a `description` that includes negative
   boundaries ("Do not use for...").
2. **When to use / when not to use** — two short lists.
3. **Inputs** — what the agent should ask for if it is missing.
4. **Procedure** — numbered, ordered, specific.
5. **Output contract** — a fenced template the agent must fill.
6. **Worked example** — one realistic input and its expected output.
7. **Common mistakes** — the failure modes you have actually hit.

Steps 5 and 6 are what separate a skill that works from a prompt that
sometimes works. Without an output contract the agent reverts to
conversational prose; without an example it guesses at the shape.

## Ecosystem Tools & Resources

Free, browser-based utilities for authoring and maintaining skills:

- **[Claude Skill Creator](https://prompttly.com/tools/claude-skill-creator)** —
  generates a `SKILL.md` package with valid YAML frontmatter from a description
  of the workflow.
- **[Prompt Optimizer](https://prompttly.com/tools/prompt-optimizer)** —
  rewrites loose prompts into structured instructions with explicit constraints
  and output contracts.
- **[How to Install and Use Claude Skills](https://prompttly.com/resources/how-to-use-claude-skills)** —
  reference guide covering directory scopes, frontmatter fields, invocation
  patterns, and verifying that a skill actually fired.

## Contributing

Issues and pull requests are welcome. A new skill should include all seven
sections listed above, and the worked example should come from a real case
rather than an invented one.

## License

[MIT](LICENSE) — use these however you like, including commercially.
