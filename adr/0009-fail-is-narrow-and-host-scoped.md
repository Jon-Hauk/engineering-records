# ADR-0009: FAIL stays narrow, container findings do not gate the exit code

- **Status:** Accepted
- **Date:** 2026-09-09
- **Deciders:** Jon Hauk

## Context

`fieldkit` findings carry one of five statuses, defined in
`fieldkit/linux/fieldkit_linux/common.py`:

```python
STATUSES = ("PASS", "FAIL", "WARN", "UNKN", "INFO")
```

**FAIL is an automation contract, not a label.** From `cli.py:228`:

```python
return 1 if any(f.Status == "FAIL" for f in findings) else 0
```

Exit `0` means no FAIL findings — including runs containing UNKN — and does
not certify security or completeness. `1` means at least one FAIL. FAIL and
WARN both render a remediation line; PASS, UNKN and INFO do not.

The container-posture module (11 checks) landed on `main` in `5e8bdd6`. It
emits **no FAIL under any condition**. Validation against a deliberately
misconfigured container on a lab host 2026-09-09 (`--privileged`,
`--pid=host`, `--restart=always`, writable `/var/run/docker.sock`, `/etc`
bound, UID 0, floating tag) confirmed all eleven checks fire correctly —
and that a container with root-equivalent access to the host scores **WARN,
the same status as an unrotated `json-file` log driver**. Evidence:
`fieldkit/linux/evidence/container-posture.md`.

FAIL is used in exactly three places in the whole Linux suite:

| Where | Condition |
|---|---|
| `hardening.py` | Neither AppArmor nor SELinux in the kernel's active LSM list |
| `hardening.py` | `sshd -T` reports `PermitRootLogin=yes` |
| `identity.py` | More than one local UID 0 account in passwd/group |

Two of those three are **configured** state carrying explicit scope caveats —
the sshd check's own detail string disclaims Match rules and running-daemon
reload state. So "we only report configured state, therefore we never FAIL"
was never the operating rule; it does not describe the existing code.

## Decision

**FAIL is reserved for conditions that are illegitimate independent of
workload, and that are stable between runs on unchanged host posture.**
Container findings emit WARN, UNKN or INFO only. No container condition gates
the exit code.

**No second severity axis is added.** A `Risk` field orthogonal to `Status`
was considered and rejected — see below.

## Why not FAIL on containers

Not because container findings are less severe. A writable Docker socket is a
path to host root and the finding says so.

**Because a gate must be reproducible, and workload state is not.** Host
configuration changes when somebody intentionally changes it. Containers start
and stop as normal operation. A container that lives five minutes would flip
the exit code to `1` and back on *identical host posture*, with nobody having
changed anything. An exit code that varies during ordinary operation is not a
gate; it is noise, and the predictable response is `|| true` in a pipeline —
which silently destroys the contract for the sshd and UID 0 cases too.

A weaker argument was considered and rejected: that FAIL is "host-level" by
nature. That is overfitting to a sample of three. All three existing FAILs are
host-scoped because until this module the tool had no workload-scoped checks
at all. Scope is a property of the sample; **reproducibility is the actual
principle.**

## Why no Risk axis

A second field (`NONE|LOW|MEDIUM|HIGH|CRITICAL`) alongside `Status` would let
a root-equivalent container outrank a log-rotation item without touching the
exit code. It is the right shape for a production tool and it was the
recommendation from adversarial review.

Rejected for now on two grounds:

1. **Half-populated is worse than absent.** Every finding in every module
   would need a value, or an unset `Risk` on the sshd `PermitRootLogin=yes`
   FAIL reads as "no risk". That makes it a whole-suite schema change, not a
   container change.
2. **The conflation is narrower than it first appears.** The status *counts*
   line cannot distinguish the two cases. A human reading the report can: the
   detail strings already say "Access to a rootful daemon is root-equivalent:
   callers can mount and modify the host filesystem" versus a logging note.
   The severity information is present; it is the machine-readable summary
   that flattens it.

This is a lab tool with no external users. The abstraction is warranted when
something consumes the summary programmatically, and not before.

## Consequences

- Container checks never affect exit status. A host running a privileged
  container with a mounted Docker socket still exits `0`. **This is a known
  and accepted gap**, not an oversight, and it is the main cost of this ADR.
- The severity scale has an unused top end within the container module. The
  counts line under-represents container risk relative to its detail text.
- Reversing this is cheap: adding FAIL conditions later is additive. Adding
  them now and removing them later would be a behaviour regression for anyone
  who had begun depending on the exit code. **Reversibility is asymmetric, and
  it points at doing nothing today.**
- If a `Risk` axis is added later, it lands across every module at once, and
  this ADR is superseded rather than amended.

## What would reopen this

- Anything begins consuming fieldkit's exit code or status counts
  programmatically — CI, a scheduled posture job, a client-facing gate.
- A container condition is identified that is illegitimate under *every*
  workload and stable across runs. Writable `/etc/shadow` inside a container
  is the strongest candidate and did not on its own justify the change.
- The tool acquires external users, at which point "lab tool" stops being the
  operative constraint.
