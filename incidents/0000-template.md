# INC-YYYY-MM-DD: One line saying what broke

- **Severity:** SEV1 | SEV2 | SEV3 | SEV4
- **Detected:** YYYY-MM-DD HH:MM — how it was noticed
- **Resolved:** YYYY-MM-DD HH:MM
- **Duration:** how long the bad state existed (often much longer than the
  time spent fixing it — say so)
- **Author:** who wrote this up

## Summary

Three sentences a stranger could read cold. What broke, what the effect was,
what fixed it.

## Impact

What was actually affected. Be specific and be honest about what you do not
know. "No evidence of X" is not the same as "X did not happen" — say which one
you mean.

## Timeline

All times local. Include the wrong turns; they are the useful part.

| Time | Event |
|---|---|
| HH:MM | ... |

## Root cause

The actual mechanism, not the trigger. Keep asking "and why did that happen?"
until you reach something you can change.

A root cause is never "human error". A human doing the obvious thing in the
situation the system created is a *system* that made the error likely.

## Why it took so long to see

Often more valuable than the root cause. What made the wrong diagnosis look
right? What signal was missing, misleading, or swallowed?

## What went well

Real entries only. Something limited the damage or shortened the diagnosis —
name it, so it does not get removed later as unnecessary.

## Action items

Concrete and assignable. "Be more careful" is not an action item.

| # | Action | Type | Status |
|---|---|---|---|
| 1 | ... | prevent / detect / mitigate | done / open |

**Prevent** — stops it recurring.
**Detect** — makes it obvious fast if it does.
**Mitigate** — reduces the damage when it does.

Aim for at least one of each. A fix with no detection just means next time is
quiet too.

## Lessons

The generalisable part. What does this say about how the system is built,
beyond this one bug?
