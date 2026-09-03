# ADR-0005: Security gates must be satisfiable, or they get bypassed

- **Status:** Accepted
- **Date:** 2026-08-11
- **Deciders:** Jon Hauk

## Context

The pre-commit config was written with a defensible set of gates: secret
scanning, SAST, linting, PowerShell analysis, large-file blocking, and
`no-commit-to-branch` protecting `main`.

On its first real execution, five of six could never have passed
([INC-2026-08-11](../incidents/INC-2026-08-11-devsecops-gates-never-executed.md)).
Two of those failures were configuration mistakes. Two were **structural**:

- **semgrep ships no Windows build.** Its hook compiles from source and aborts.
  On this machine it can only ever fail.
- **`no-commit-to-branch` blocked `main`** in a solo repo with no PR flow. Every
  commit would fail.

The tempting response to a failing hook is `git commit --no-verify`. That flag
is not selective — it disables *every* hook, including gitleaks. So a gate that
cannot be satisfied does not merely fail to protect; it trains the operator
into a habit that disables the gates that work.

## Decision

Every gate that runs locally must be **satisfiable on the developer machine**.
A check that cannot pass here belongs in CI, or nowhere.

Applied:

| Gate | Placement | Why |
|---|---|---|
| gitleaks, ruff, hygiene hooks | local + CI | run everywhere |
| PSScriptAnalyzer | local only | needs Windows PowerShell |
| semgrep | **CI only** | no Windows build exists |
| `no-commit-to-branch` | **removed** | no PR flow to land commits instead |
| whole-config run | CI | catches a hook config that nobody installed |

CI is where enforcement lives, because it cannot be bypassed. Local hooks are
for fast feedback, and are deliberately skippable.

## Alternatives considered

**Option A — keep semgrep local via WSL or Docker.** Technically possible.
Rejected: it adds a heavy dependency to every commit on a machine with neither
installed, to run a scan that CI already performs on push.

**Option B — keep `no-commit-to-branch` and adopt branch-and-PR.** Defensible,
and correct for a team. Rejected for now as ceremony without a reviewer: a
solo operator opening PRs to approve them alone gets the friction and none of
the benefit. Revisit when a second person commits here.

**Option C — leave the failing hooks and use `--no-verify` when needed.**
Rejected — this is the failure mode being designed against. It is also
indistinguishable from Option A on the day you are in a hurry.

## Consequences

**Good**
- Every local gate passes on a clean tree, so a failure means something.
- `--no-verify` stays exceptional rather than routine.
- Platform constraints are recorded next to the config, so the next person does
  not "helpfully" restore semgrep locally and rediscover it.

**Bad / accepted costs**
- SAST feedback moves from commit-time to push-time.
- `main` is unprotected against direct commits. Accepted while solo; it is the
  first thing to revisit when that changes.
- Two enforcement locations to keep in step.

**Neutral**
- The local hook set is smaller than a maximal config would suggest. Smaller
  and true beats larger and decorative.

## What would change this decision

A second contributor changes the `no-commit-to-branch` calculus immediately —
at that point branch protection and PR review earn their friction.

If semgrep ships a Windows build, move it back to local.

## References

- [`pipeline/pre-commit-config.yaml`](../pipeline/pre-commit-config.yaml)
- [`pipeline/ci.yml`](../pipeline/ci.yml) — `pre-commit`, `sast`
- [INC-2026-08-11](../incidents/INC-2026-08-11-devsecops-gates-never-executed.md)
