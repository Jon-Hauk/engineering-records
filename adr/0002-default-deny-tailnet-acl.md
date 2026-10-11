# ADR-0002: Default deny on the tailnet, with no wildcard fallback

- **Status:** Accepted
- **Date:** 2026-08-10
- **Deciders:** Jon Hauk

## Context

Tailscale ships a default policy equivalent to:

```hujson
{"acls": [{"action": "accept", "src": ["*"], "dst": ["*:*"]}]}
```

Every device reaches every other device on every port. This was the live state
of `<tailnet>.ts.net` (4 devices: 2 Windows, 1 Linux, 1 Android).

The threat this leaves open is not "someone breaches the tailnet" — WireGuard
and the identity layer are strong. It is **lateral movement after a single
endpoint is compromised**. A phone that installs a malicious app, or a laptop
that runs the wrong installer, inherits full reach to everything else. The
overlay network faithfully carries the attack to every host, including from
untrusted networks where the LAN firewall would never have applied.

A SOHO network's real weakness has never been the perimeter. It is that
everything inside is implicitly trusted, and the flat network does the
attacker's lateral movement for them.

## Decision

We will run a **default-deny** policy in which every reachable port is
enumerated, and **no rule anywhere grants `*` to a device**.

Removing a grant removes access. There is no broader rule underneath it to
fall back on. The absence of a rule *is* the control, and
`acl.hujson` (in the private lab repo) is written so that absence is
visible: `tag:iot` is declared in `tagOwners` and then appears in no grant, as
a source or a destination, on purpose.

## Alternatives considered

**Option A — keep allow-all, rely on host firewalls.** Zero policy to
maintain, and each host defends itself. Rejected: it distributes the control
across four machines with three operating systems, one of which (Android)
gives you effectively no firewall control. It also fails silently — you cannot
audit "is lateral movement blocked?" from one place, and you will never notice
when one host drifts.

**Option B — deny-by-default but with a broad `group:admin → *` for
convenience, applied to devices.** Rejected because it defeats the point.
Nearly every device here is admin-owned, so a device-scoped wildcard restores
allow-all under a different name.

**Option C — segment by subnet instead of by identity.** This is the
traditional VLAN approach. Rejected: it re-introduces "location implies
trust", which is exactly what zero trust removes, and it does not work at all
for devices roaming onto hotel wifi — the case where you need it most.

## Consequences

**Good**
- One compromised endpoint reaches only what its tag explicitly allows.
- The policy is auditable by reading one file.
- Adding a service is a deliberate act with a reviewable diff.

**Bad / accepted costs**
- Every new service needs a policy change; forget, and it silently fails to
  connect. This is the intended friction, but it *is* friction.
- Enumerated ports rot. A service that moves ports breaks until the ACL
  catches up.
- Real lockout risk while applying. Mitigated by ordering (see ADR-0003) and
  by keeping an authenticated admin console tab open during rollout.

**Neutral**
- The file is longer than the default. That is the trade.

## What would change this decision

If the tailnet grows past roughly a dozen services, enumerating ports by hand
stops scaling and this should move to generated policy with `autogroup` and
per-service tags rather than a hand-maintained list.

If Tailscale ever ships per-service identity (rather than per-device), the tag
layer here becomes partly redundant.

## References

- `zta-soho/policy/acl.hujson` (in the private lab repo)
- [ADR-0003](0003-device-tags-hold-less-authority.md) — why tags, and the
  ordering constraint when applying this
- [ADR-0004](0004-policy-as-code-tested-in-ci.md) — how it is tested
