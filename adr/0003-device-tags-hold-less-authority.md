# ADR-0003: Device tags hold strictly less authority than their owner

- **Status:** Accepted
- **Date:** 2026-08-10
- **Deciders:** Jon Hauk

## Context

In Tailscale, an untagged device is **owned by the user who enrolled it** and
inherits that user's grants. Every device on `<tailnet>.ts.net` was untagged
and owned by `<owner>@github`, who is the sole admin.

Combined with the admin grant in [ADR-0002](0002-default-deny-tailnet-acl.md),
that means malware executing on any machine holds *the same authority as the
human sitting at the keyboard*. Default-deny does not help here: the deny is
scoped by identity, and the malware has the identity.

This is the gap between "I have an ACL" and "I have zero trust". Identity-based
access alone still hands full reach to anything running as that identity.

## Decision

Devices are assigned tags (`tag:workstation`, `tag:server`, `tag:mobile`,
`tag:iot`, `tag:ci`, `tag:exit`). A tagged node is owned by the **tag**, not
the person, and every tag receives an explicitly enumerated subset of what the
human gets.

Concretely: `<owner>@github` gets `*`. `tag:workstation` gets ports 22, 3389
and 445 to peers, and a named list to servers. It never gets `*`, even though
its owner does.

The human keeps the break-glass path. The machine does not.

## Alternatives considered

**Option A — tags that mirror the user's access.** Simple, no capability lost.
Rejected outright: it is the current state renamed, and provides nothing.

**Option B — no tags, use device posture instead.** Posture checks OS version
and Tailscale version, which is real defence, but it verifies the device is
*healthy*, not that it is *limited*. A fully patched laptop running malware
passes every posture check. Posture is complementary, not a substitute — it is
used here as well, where the plan allows it.

**Option C — separate user accounts per device class.** Achieves similar
separation via identity rather than tags. Rejected: it multiplies accounts to
manage, breaks SSO cleanly, and Tailscale's tag mechanism exists precisely for
this.

## Consequences

**Good**
- Highest-leverage control in the whole policy. Code running as the machine
  holds strictly less than the operator does.
- Makes blast radius explicit and reviewable per device class.
- `tag:ci` matters disproportionately: build runners hold credentials and are
  a prime lateral-movement target, and they are confined to 443 on servers.

**Bad / accepted costs**
- **Tagging a device removes your user-level reach from it.** After tagging,
  `workstation-a` stops matching `group:admin` and gets only `tag:workstation` grants.
  This surprises people; it is the control working.
- Tagging requires re-authentication per device.
- Reversing a tag needs another re-auth.

**Neutral**
- Tag assignment happens at enrolment: `tailscale up --advertise-tags=tag:workstation`

## Ordering constraint — learned the hard way

**Apply the ACL first, tag devices second.** This is the opposite of the
intuitive order and it was verified by breaking it.

A node may only advertise a tag that already exists in `tagOwners` in the
**live** policy. Advertising an unknown tag causes Tailscale to invalidate the
node's authorisation: the backend drops to `NeedsLogin` and the machine falls
off the tailnet. This happened to `workstation-a` on 2026-08-10 while the policy
existed only as an unapplied file.

Recovery: `tailscale up --advertise-tags=""`

Applying the ACL first is safe, because untagged devices are still owned by
`<owner>@github` and keep working via the admin grant through the transition.

## What would change this decision

If a second human joins the tailnet, `group:household` needs real grants and
the tag scheme should be revisited so tags compose with multiple identities
rather than assuming one admin.

## References

- `zta-soho/policy/acl.hujson` (in the private lab repo) — `tagOwners`
- `zta-soho/scripts/posture-check.ps1` (in the private lab repo)
  — reports untagged as a failing control
- [ADR-0002](0002-default-deny-tailnet-acl.md)
