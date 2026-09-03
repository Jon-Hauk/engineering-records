# ADR-0001: Record architecture decisions

- **Status:** Accepted
- **Date:** 2026-08-11
- **Deciders:** Jon Hauk

## Context

This workspace holds a live zero trust deployment, a DevSecOps pipeline, and a
development toolchain. Most of the security decisions in it look arbitrary or
even wrong from the outside, because the reasoning is invisible:

- `tag:iot` is defined in `tagOwners` and appears in no grant. Looks like an
  unfinished config. It is the control.
- `no-commit-to-branch` is deliberately absent from the pre-commit hooks.
  Looks like a missing best practice. It was removed for a reason.
- SSH uses `check` with a 12h period rather than `accept`. Looks like
  friction someone forgot to turn off.

The operator is one person learning the domain. Six weeks from now that person
has no memory of the trade-offs, and the config is indistinguishable from
something copied off a blog.

## Decision

We will record every non-obvious security or workflow decision as a numbered
ADR in `adr/`, using [the template](0000-template.md).

ADRs are immutable once Accepted. A reversal is a new ADR that supersedes the
old one; the old one stays, marked, because the wrong turn is part of the
record.

## Alternatives considered

**Option A — comments in the config.** Already doing this, and it is not
enough. A comment explains what a line does. It has no room for the three
options rejected, the threat model, or the plan-tier constraint that shaped
it. `acl.hujson` would become unreadable if it carried that weight.

**Option B — a wiki or Notion.** Drifts from the code immediately, because
nothing forces them to move together. Docs that live outside the repo are
updated when someone remembers, which is never.

**Option C — nothing, rely on memory.** Works for about two weeks. It is also
exactly what separates "configured some security tools" from security
engineering when someone asks why.

## Consequences

**Good**
- Decisions survive the operator forgetting them.
- Reviewing a change becomes possible: the diff shows what, the ADR shows why.
- Builds the reasoning habit deliberately — arguing yourself out of the two
  rejected options is where the learning is.
- Portfolio value. The record of judgement is worth more than the config.

**Bad / accepted costs**
- ~15 minutes per decision.
- Tempting to over-document. Not every choice needs one; if nobody would ever
  argue it, skip it.
- Stale ADRs mislead. Mitigated by superseding rather than editing.

**Neutral**
- Numbering is permanent. Gaps are fine.

## What would change this decision

If this becomes a team and a real design-doc process exists, ADRs may be
redundant with it. Until then, they are the lightest thing that works.

## References

- [docs/README.md](../README.md) — the practice
- Michael Nygard's original ADR pattern
