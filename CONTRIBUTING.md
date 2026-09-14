# Contributing

Contributions are welcome. This repo is small and opinionated — the bar is that
a skill should be one you actually use, not one that seems like it would be
useful.

## Adding a skill

Create `skills/<kebab-case-name>/SKILL.md`. The directory name and the
frontmatter `name` must match.

Every skill needs all seven sections:

1. **Frontmatter** — `name` and `description`. The description is what Claude
   Code matches against at runtime, so it must say both what the skill does and
   when *not* to use it. Under 1,024 characters.
2. **When to use this skill** — three to five concrete situations.
3. **When not to use it** — the cases where it will produce noise. This is not
   optional; a skill without negative boundaries over-triggers.
4. **Inputs** — what the agent should ask for if it is missing, rather than
   guessing.
5. **Procedure** — numbered and ordered. Say what to check, not "analyse the
   code".
6. **Output contract** — a fenced template the agent fills in. Without one the
   agent reverts to conversational prose and the skill stops being reusable.
7. **Worked example** — one realistic input and its expected output.

A "Common mistakes" section at the end is strongly encouraged. Write the
failure modes you actually hit, not hypothetical ones.

## Style

- Write for someone who will read the skill to understand it, not just install
  it. These files are documentation as much as configuration.
- Prefer concrete thresholds over adjectives. "Under 1,024 characters" beats
  "keep it short".
- No emoji in `SKILL.md`.
- Examples should be plausible code, not `foo` and `bar`.

## Validating frontmatter

Before opening a PR:

```bash
python3 - <<'PY'
import re, glob
ok = True
for f in sorted(glob.glob('skills/*/SKILL.md')):
    t = open(f).read()
    m = re.match(r'^---\n(.*?)\n---\n', t, re.S)
    if not m:
        print('FAIL no frontmatter:', f); ok = False; continue
    fm, slug = m.group(1), f.split('/')[1]
    name = re.search(r'^name:\s*(.+)$', fm, re.M).group(1).strip()
    desc = re.search(r'^description:\s*(.+)$', fm, re.M).group(1).strip()
    kebab = bool(re.fullmatch(r'[a-z0-9]+(-[a-z0-9]+)*', name))
    good = name == slug and kebab and len(desc) < 1024 and 'Do not use' in desc
    ok = ok and good
    print(('OK ' if good else 'BAD'), slug, f'desc {len(desc)}/1024')
print('ALL VALID' if ok else 'VALIDATION FAILED')
PY
```

## Testing a skill

Install it locally and confirm it actually fires:

```bash
cp -r skills/<name> ~/.claude/skills/
```

Start a new Claude Code session, then check both invocation paths:

- Explicit: `/<name> <task>`
- Semantic: describe the task in your own words and confirm the skill is
  selected without naming it

A skill that only works when invoked explicitly has a description problem.
Verify the output matches the skill's own output contract — if the response is
conversational prose, the contract is not specific enough.

## Pull requests

One skill per PR. Include in the description which projects you used it on and
what it caught — that is the only real evidence a skill works.
