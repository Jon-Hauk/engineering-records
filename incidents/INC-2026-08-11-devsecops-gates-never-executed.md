# INC-2026-08-11: DevSecOps gates existed for a day without ever executing

- **Severity:** SEV3 — no breach, but every security gate in the repo was
  inert while appearing configured
- **Detected:** 2026-08-11 ~07:20, on the first commit after `pre-commit install`
- **Resolved:** 2026-08-11 ~07:55
- **Duration:** ~19 hours of false assurance (config committed 2026-08-10 ~12:30)
- **Author:** Jon Hauk

## Summary

A `.pre-commit-config.yaml` with secret scanning, SAST, linting and PowerShell
analysis was written and committed, but the git hooks were never installed, so
none of it ran. When the hooks were finally installed, the very first real
invocation failed closed and revealed that **five of the six gates could never
have worked** — wrong pinned versions, a tool with no Windows build, a missing
module, and a hook that blocked the only branch in use.

Nothing was breached. The failure was that the repo *looked* protected for a
day while being completely unprotected.

## Impact

- Secret scanning (gitleaks) did not run on the initial commit of 32 files.
  That commit was manually scanned for credential patterns before it landed
  and was clean, so **there is no evidence any secret was committed** — but
  that was luck and a manual check, not the control working.
- SAST (semgrep) never ran locally and still cannot on this machine.
- PowerShell scripts that handle API credentials were never linted.
- CI would have caught the secret and SAST cases on push, but nothing had
  been pushed — no remote was configured.

## Timeline

| Time | Event |
|---|---|
| 08-10 12:30 | `.pre-commit-config.yaml` written and committed. Versions written from memory, not checked. |
| 08-10 12:35 | `pre-commit validate-config` run — **passes**. Only checks YAML shape, not that the referenced revisions exist. |
| 08-10 ~21:00 | First real commit. `pre-commit install` had not been run, so no hooks fired. Nothing indicated this. |
| 08-11 07:15 | `pre-commit install` finally run. |
| 08-11 07:20 | First hooked commit fails: semgrep `v1.105.0` — `pathspec did not match any file(s) known to git`. |
| 08-11 07:25 | Checked real tags via `git ls-remote`. All four pins were wrong; latest were `v1.172.0`, `v8.30.1`, `v0.16.2`, `v6.0.0`. |
| 08-11 07:35 | With versions fixed, semgrep still fails — it tries to compile from source. Semgrep ships no Windows build. |
| 08-11 07:40 | `no-commit-to-branch` fails: it blocks `main`, which is the only branch. |
| 08-11 07:45 | PSScriptAnalyzer fails — module never installed. Install fails again: `PowerShellGet` needs the NuGet provider and prompts, but the shell is non-interactive. |
| 08-11 07:50 | Module installs after bootstrapping the NuGet provider explicitly. Still fails to *load* — execution policy blocks the module's own scripts. |
| 08-11 07:52 | Fixed with `-ExecutionPolicy Bypass`. Also found `-Path . -Recurse` would scan the 4 GB gitignored `comfyui/` clone on every commit. |
| 08-11 07:55 | All gates pass. The first commit ever actually scanned lands. |

## Root cause

Two independent causes that combined to hide each other:

1. **Writing the config is not installing the gate.** `pre-commit` requires
   `pre-commit install` to write `.git/hooks/pre-commit`. Committing the YAML
   changes nothing. The repo had the *description* of a gate, not a gate.

2. **The config was never executed before being trusted.** Every version pin
   was written from memory and four of five were wrong. `validate-config`
   passed, which created false confidence: it validates YAML structure, not
   that the pinned revisions resolve or that the tools can run on this
   platform.

Underneath both: **a control was assumed to work because it was written
down.** No step in the process required it to run successfully once.

## Why it took so long to see

- `pre-commit validate-config` returning success was actively misleading. It
  answers "is this valid YAML in the right shape?" and was read as "is this
  configured correctly?"
- There is no negative signal when hooks are absent. A commit with no hooks
  installed looks exactly like a commit where every hook passed. Silence
  reads as success.
- The failure was in the *meta* layer. The security tools were all real, the
  rules were sensible; the thing that was broken was whether any of it
  executed, which nothing was watching.

## What went well

- The gates **failed closed**. When they finally ran, they refused to let the
  commit through rather than warning and continuing. Five defects surfaced in
  thirty minutes because nothing was skippable.
- The initial commit had been manually scanned for credential patterns before
  landing, which is why the gitleaks gap caused no actual exposure.
- `.gitignore` was correct independently, so `.env` was never staged even
  with gitleaks inert. Defence in depth worked by accident.

## Action items

| # | Action | Type | Status |
|---|---|---|---|
| 1 | Run `pre-commit install`; hooks now active | prevent | done |
| 2 | Correct all four version pins to tags verified via `git ls-remote` | prevent | done |
| 3 | Move semgrep to CI only — no Windows build exists | prevent | done |
| 4 | Install PSScriptAnalyzer; add `-ExecutionPolicy Bypass` and scope paths | prevent | done |
| 5 | Remove `no-commit-to-branch` — see Lessons | prevent | done |
| 6 | Add `pre-commit run --all-files` to CI so a broken hook config fails the build even if nobody installed hooks locally | **detect** | done |
| 7 | Configure a git remote and push, so CI runs at all | **detect** | open |
| 8 | Add `pre-commit autoupdate` to a monthly routine so pins do not rot | mitigate | open |

Items 6 and 7 are the important ones. Everything above them fixes *this*
occurrence; only 6 and 7 would catch the next one.

**Item 7 was the single largest gap in the lab repo at the time.** Every
detection control there — the CI pre-commit job, gitleaks over full history,
semgrep, trivy, the ACL access tests — runs on push. No remote was
configured, so none of it had ever run. The detection layer was in exactly
the state the prevention layer had been in the day before: written down,
never executed.

## Lessons

**A control that has never executed is not a control.** It is a description of
one. The test for "is this configured?" is running it and watching it fail on
purpose — not reading the config and agreeing with it.

**Validation that checks shape gets mistaken for validation that checks
function.** `validate-config` passing felt like assurance. Whenever a tool
tells you something is valid, ask precisely which property it verified.

**A gate that always fails gets bypassed, and then you have no gate.**
`no-commit-to-branch` blocking `main` in a repo with no PR flow would have
failed every single commit. The realistic outcome is not "I will now adopt
branches", it is `--no-verify` becoming muscle memory — which disables *every*
hook, including the secret scanner. Security friction that cannot be satisfied
trains people to disable security. Removing that hook made the repo safer.

**Absence of failure is not evidence of success** when the mechanism is silent
by design. Any check whose success looks identical to its absence needs a
positive signal — which is what action item 6 provides.
