# Architecture Decision Records

Why the security and workflow choices are what they are.

| # | Decision | Status | Date |
|---|---|---|---|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions | Accepted | 2026-08-11 |
| [0002](0002-default-deny-tailnet-acl.md) | Default deny on the tailnet, with no wildcard fallback | Accepted | 2026-08-10 |
| [0003](0003-device-tags-hold-less-authority.md) | Device tags hold strictly less authority than their owner | Accepted | 2026-08-10 |
| [0004](0004-policy-as-code-tested-in-ci.md) | The tailnet policy is code, and it is tested like code | Accepted | 2026-08-10 |
| [0005](0005-security-gates-must-be-satisfiable.md) | Security gates must be satisfiable, or they get bypassed | Accepted | 2026-08-11 |
| [0006](0006-actions-that-spend-money-require-confirmation.md) | Actions that spend money require explicit confirmation | Accepted | 2026-08-10 |
| [0007](0007-blender-bridge-is-localhost-only.md) | The Blender bridge binds localhost only, with a token, off by default | Accepted | 2026-08-10 |
| [0009](0009-fail-is-narrow-and-host-scoped.md) | FAIL stays narrow; container findings do not gate the exit code | Accepted | 2026-09-09 |

Gaps in the numbering are ADRs held back with the lab — topology, host
posture, anything that does not stand alone here. A number is allocated
once and never reused. The tailnet ADRs (0002-0004) are published with the
tailnet name, account identity and device names replaced by placeholders,
and links into the private policy repository turned into plain references.

[`0000-template.md`](0000-template.md) is the blank.

**Publication boundary:** ADR 8 must never be published, including a redacted
copy (JH 20261018). The ignore rules cover its usual filenames as a guard
against accidental addition; they are not a substitute for reviewing content
and history. Never copy the private ADR directory wholesale into this repo.

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
