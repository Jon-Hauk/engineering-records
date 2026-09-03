# Architecture Decision Records

Why the security and workflow choices are what they are.

| # | Decision | Status | Date |
|---|---|---|---|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions | Accepted | 2026-08-11 |
| [0005](0005-security-gates-must-be-satisfiable.md) | Security gates must be satisfiable, or they get bypassed | Accepted | 2026-08-11 |
| [0006](0006-actions-that-spend-money-require-confirmation.md) | Actions that spend money require explicit confirmation | Accepted | 2026-08-10 |
| [0007](0007-blender-bridge-is-localhost-only.md) | The Blender bridge binds localhost only, with a token, off by default | Accepted | 2026-08-10 |

Gaps in the numbering are ADRs held back with the lab — topology, host
posture, anything that does not stand alone here. A number is allocated
once and never reused.

[`0000-template.md`](0000-template.md) is the blank.

## Verification markers

Claims in later ADRs carry their evidence status, because three
controls were asserted as working and broke on first execution:

| Marker | Meaning |
|---|---|
| **[VERIFIED]** | A named command was run that establishes this |
| **[UNVERIFIED]** | Asserted from reasoning. A claim, not a fact. |
| **[ASSUMED]** | Depends on operator statement or unobservable external state |

A reviewer should read the [UNVERIFIED] and [ASSUMED] lines first. They are
where the document is weakest, which is the point of marking them.
