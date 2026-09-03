# Incidents

What broke, why, and what changed as a result.

| ID | Title | Sev | Date |
|---|---|---|---|
| [INC-2026-08-11](INC-2026-08-11-devsecops-gates-never-executed.md) | DevSecOps gates existed for a day without ever executing | SEV3 | 2026-08-11 |
| [INC-2026-08-11](INC-2026-08-11-download-resume-never-worked.md) | Downloads never resumed, and the recovery made it worse | SEV4 | 2026-08-11 |

Both entries so far share a theme worth noticing: a control that was believed
to work, written down as working, and never once observed working.

## Severity

Judgement, not a formula. Pick the one you would defend.

| | Meaning |
|---|---|
| **SEV1** | Confirmed compromise, data loss, or credential exposure outside your control |
| **SEV2** | A security control failed open, or a secret was exposed somewhere recoverable |
| **SEV3** | A control was ineffective or absent, but no evidence of exploitation |
| **SEV4** | Degraded or noisy, no security impact |

## Writing one

```bash
cp 0000-template.md INC-2026-08-15-short-slug.md
```

Write it while it is fresh — within a day. The details that matter are the
ones you will forget: what you *thought* was happening, and why that looked
right.

## Blameless means blameless

Not a formality. Self-inflicted incidents are the most instructive ones in
this repo, and they only get written honestly if writing them costs nothing.

The rule: **a root cause is never "human error".** If a person did the obvious
thing and it broke, the system made that outcome likely. Ask what the system
did to invite it. "I forgot to run `pre-commit install`" is a trigger; the
cause is that nothing anywhere required the gate to run successfully once.

## Write one whenever you were surprised

Not just for outages. The bar is *surprise*, because surprise means your model
of the system was wrong, and that is worth the twenty minutes regardless of
whether anything was damaged.

Especially write one when you caused it yourself. Especially when the fix was
embarrassing.

## Action items are the point

An incident with no action items is a diary entry. Every write-up should
produce at least one item of each kind:

- **Prevent** — stops this exact thing recurring
- **Detect** — makes it loud and fast if it does
- **Mitigate** — limits the damage when it does

Teams consistently over-invest in *prevent* and skip *detect*. Prevention
addresses the failure you already understand. Detection catches the one you
have not thought of yet, which is every future incident.
