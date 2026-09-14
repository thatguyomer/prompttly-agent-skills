---
name: incident-postmortem
description: Drafts a blameless incident postmortem from logs, a timeline, and commit history. Separates trigger from root cause, builds a minute-by-minute timeline with detection and mitigation times, identifies contributing factors across code, process, and monitoring, and produces action items with owners rather than vague resolutions. Use after an outage, degradation, or near miss, or when preparing an incident review document. Do not use for live incident response while a system is still down, for performance analysis without an incident, or for individual performance review.
---

# Incident Postmortem

Produces a postmortem that makes the system more reliable rather than one that
assigns blame and gets filed away. Blameless means assuming everyone acted
reasonably given what they knew at the time — so the question is what the
system let them believe.

## When to use this skill

- After an outage, degradation, or data incident is resolved
- After a near miss that only luck prevented from becoming an outage
- Preparing for an incident review meeting

## When not to use it

- During a live incident — restore service first, this is a distraction
- General performance investigation with no incident attached
- Anything touching individual performance evaluation. If the output would
  influence someone's review, this is the wrong document

## Inputs

Ask for whatever is missing; do not invent facts to fill the template:

1. Rough timeline: when it started, when someone noticed, when it was fixed
2. Logs, alerts, dashboards from the window
3. Deploys and commits in the preceding 48 hours
4. Customer impact: who, how many, what could they not do
5. What responders tried, including what did not work

## Trigger versus root cause

The trigger is what happened immediately before. The root cause is the
condition that made the trigger capable of causing an outage.

> "The deploy caused it" is a trigger. "A config change could reach production
> without any validation of the connection-pool ceiling" is a root cause.

Keep asking "and why was that possible?" until you reach something you can
change. Stop when the next answer would be about a person rather than a system.

## Procedure

1. **Build the timeline** in UTC, one line per event, marking:
   - `T0` — when the fault was introduced (often earlier than the impact)
   - `T_impact` — when users were first affected
   - `T_detect` — when a human knew
   - `T_mitigate` — when impact stopped
   - `T_resolve` — when the underlying cause was fixed
2. **Compute the gaps.** `T_detect - T_impact` is your detection gap and is
   usually the most actionable number in the document. `T_mitigate - T_detect`
   is your response gap.
3. **Quantify impact** in user terms: requests failed, users affected, data
   lost, revenue. "Elevated error rate" is not an impact statement.
4. **Separate trigger from root cause**, then list contributing factors in
   three buckets: code, process, monitoring.
5. **Write action items** that are specific, owned, and verifiable. "Improve
   monitoring" is not an action item. "Alert when connection pool utilisation
   exceeds 80% for 5 minutes — owner: X" is.
6. **Note what went well.** Fast rollback, good runbook, clear escalation.
   Postmortems that only list failures train people to avoid writing them.

## Output contract

```
## Summary
<2-3 sentences: what broke, for whom, for how long>

## Impact
- Duration: <T_impact to T_mitigate>
- Affected: <quantified>
- Data loss: <yes/no, scope>

## Timeline (UTC)
| Time | Event | Source |

## Detection and response
- Detection gap: <T_detect - T_impact> — <how it was detected>
- Response gap: <T_mitigate - T_detect>

## Trigger
<the immediate cause>

## Root cause
<the condition that allowed the trigger to cause impact>

## Contributing factors
- Code: ...
- Process: ...
- Monitoring: ...

## What went well
- ...

## Action items
| Action | Owner | Priority | Verification |

## Open questions
- <anything unknown, rather than guessed>
```

Populate "Open questions" honestly. A postmortem with a confident root cause
that was actually a guess is worse than one that names its uncertainty.

## Worked example (abbreviated)

**Input:** checkout 500s for 40 minutes after a Tuesday deploy.

```
## Trigger
A deploy at 14:02 raised per-instance DB connections from 10 to 25.

## Root cause
The connection-pool size was configurable per service with no validation
against the database's global max_connections (200). At 12 instances the new
setting requested 300 connections, exhausting the limit and starving every
other service. Nothing in CI, code review, or deploy tooling could detect that
a per-service value had become globally unsafe.

## Contributing factors
- Code: pool size read from env with no upper bound or cross-service awareness.
- Process: the change was reviewed by one person, on a config file with no
  listed owner; the reviewer had no way to know the global ceiling.
- Monitoring: connection-pool utilisation was collected but had no alert. The
  first signal was customer reports, 11 minutes after impact.

## Action items
| Action | Owner | Priority | Verification |
| Alert at 80% of max_connections for 5m | — | P0 | Fire a test alert |
| Validate total requested connections in CI | — | P1 | CI fails a PR that exceeds the ceiling |
| Add CODEOWNERS for infra config | — | P2 | PR requires infra review |
```

## Common mistakes

- Stopping at the trigger. "Bad deploy" is where the analysis starts.
- Action items with no owner, which are wishes.
- Writing "human error" as a root cause. Ask what the system let that human
  believe, and why it was possible to act on it.
- Omitting the detection gap, which is usually the cheapest thing to improve.
