# ADR-0004: The tailnet policy is code, and it is tested like code

- **Status:** Accepted
- **Date:** 2026-08-10
- **Deciders:** Jon Hauk

## Context

Tailscale's admin console has a web editor for the ACL. Editing there is
immediate, requires no tooling, and is the path of least resistance.

It also means the security policy of the entire network has no history, no
review step, and no test. A change that accidentally lets a phone SSH into a
server is indistinguishable from a change that fixes a typo — both are a
person clicking Save.

The policy in this repo is default-deny with enumerated ports
([ADR-0002](0002-default-deny-tailnet-acl.md)) and a tag scheme where the
absence of a rule *is* a control ([ADR-0003](0003-device-tags-hold-less-authority.md)).
That kind of policy fails silently: delete the wrong three lines and nothing
errors, nothing alerts, and lateral movement is quietly re-enabled.

## Decision

The ACL lives in `zta-soho/policy/acl.hujson`, in git, and carries its own
`tests` block asserting reachability in both directions:

```hujson
{ "src": "tag:mobile", "deny": ["server-a:22", "workstation-a:22"] },
{ "src": "tag:iot",    "deny": ["server-a:443", "workstation-a:22"] },
```

CI validates the policy and runs those tests against Tailscale's own evaluator
on every push. Applying it is a script (`scripts/apply-acl.ps1`), never a
console paste, and the apply path validates before it writes.

## Alternatives considered

**Option A — edit in the admin console.** Fastest, zero tooling. Rejected: no
diff, no review, no test, no rollback beyond Tailscale's own history. For a
control whose failure mode is silence, that is not enough.

**Option B — version the file in git but apply by hand.** Gets history and
review. Rejected as a half-measure: nothing forces the file and the live
policy to agree, so they drift, and the file becomes documentation of what
someone once intended.

**Option C — full IaC (Terraform provider).** Real state management and drift
detection. Rejected as disproportionate for four devices and one operator. The
Tailscale API plus a validate step covers the same ground here. Revisit if the
tailnet grows.

## Consequences

**Good**
- A policy change is a reviewable diff.
- The `tests` block encodes intent that the grants alone do not express.
  "`tag:iot` must not reach anything" is asserted, not merely implied by
  omission — so the control survives a future edit that does not understand it.
- A bad policy fails the build rather than the network.
- Rollback is `git revert` plus re-apply.

**Bad / accepted costs**
- Slower than clicking Save.
- Requires a `TS_API_KEY` with `acl:write` in CI — a powerful credential,
  scoped and expiring, but real.
- CI can only run these tests if a remote exists. As of writing, none does, so
  **the tests have never executed** — the same failure documented in
  [INC-2026-08-11](../incidents/INC-2026-08-11-devsecops-gates-never-executed.md).

**Neutral**
- The `tests` block grows with the policy.

## What would change this decision

If the tailnet grew past a dozen services or gained a second operator,
Terraform's state and plan/apply model would start earning its complexity.

## References

- `zta-soho/policy/acl.hujson` (in the private lab repo) — `tests` block
- `zta-soho/scripts/apply-acl.ps1` (in the private lab repo)
- [`.github/workflows/ci.yml`](../pipeline/ci.yml) — `zta-policy` job
