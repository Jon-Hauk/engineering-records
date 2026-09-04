# Reading a pipeline

How to tell whether CI is doing anything, without reading the YAML first.

Writing a workflow is the cheap half — the syntax is looked up, not recalled.
The half that cannot be delegated is answering **"did this actually do the
thing it claims?"** Every failure below is one this repo has already made, and
they are ordered by how cheap the check is. The first three need no
understanding of what the workflow does.

---

## The exhibit

```
✓ zta-policy    6s
```

Green on every run it ever had. The repository secret it depends on had never
been set, so the tailnet ACL was never once validated — while the job sat
directly under a comment claiming that a change letting a phone SSH into a
server would fail here.

The config was real. The tool was real. The rules were sensible. What was
broken was whether any of it *executed*, and nothing was watching that.

---

## 1. Duration that does not match the claim

Start here, because it needs no file open at all. A job that reaches an API,
downloads a toolchain or scans a repository has a floor on how long it can
take. Runtime far below that floor means the work did not happen.

| | |
|---|---|
| **Reads as** | Fast, clean, well-optimised pipeline |
| **Actually** | Six seconds is checkout plus an `if`. A round trip to a remote API does not fit inside it. |

The same tell shows up outside CI. A GPU discovery that aborts in 1.4 s and
falls back to CPU, where a healthy one takes twelve, reports no error at all —
everything simply runs ten times slower until a human notices.

**The check:** open the run list. For each job ask *what is the shortest this
could honestly take?* Anything conspicuously under that is doing less than it
says.

---

## 2. The escape hatch

What made those six seconds possible. A step that exits zero when its
precondition is missing converts "could not run" into "passed."

```yaml
# the version that was green for months
run: |
  if [ -z "$TS_API_KEY" ]; then
    echo "::warning::TS_API_KEY not set — skipping live ACL validation."
    exit 0          # <-- the gate's off switch, permanently flipped
  fi
  curl -sSf ... /acl/validate
```

A `::warning::` does not fail a build. It is a grey note nobody reads on a run
that is already green.

| | |
|---|---|
| **Reads as** | Graceful degradation around an optional secret |
| **Actually** | The secret was never optional and was never set. The bypass was the only path the job ever took. |

The family: `exit 0`, `|| true`, `continue-on-error: true`, `if:` conditions
that quietly evaluate false, and a test step that passes having collected zero
tests.

**The check:** grep the workflow for those. For each hit answer one question —
**what makes this step green without doing its job?** If the answer is "a
missing secret", "a file that is not there" or "a tool that is not installed",
that is not error handling. That is the off switch, and it may already be
flipped.

---

## 3. Written, but never installed

The original failure, recorded in full at
[INC-2026-08-11](../incidents/INC-2026-08-11-devsecops-gates-never-executed.md).
A `.pre-commit-config.yaml` carrying secret scanning, SAST, linting and
PowerShell analysis was committed at 12:30. `pre-commit install` was not run
until 07:15 the next morning. For nineteen hours the repo held the
*description* of six gates and not one gate.

| | |
|---|---|
| **Reads as** | Commits landing cleanly, no complaints from any hook |
| **Actually** | No hooks installed. A commit with zero hooks looks byte-for-byte identical to one where all six passed. |

This is the one that generalises furthest, because it has **no negative
signal**. Silence reads as success. Same shape as an empty secret store, and
same shape as a workflow sitting in a repo with no remote configured — where
every push-triggered control had never run even once.

**The check:** look for the run, not the file. Does `.git/hooks/pre-commit`
exist? Does the Actions tab show a run for this workflow, on this branch,
recently? **No run means no gate**, however good the YAML is.

---

## 4. A green check that verified a different property

Before that first commit, `pre-commit validate-config` was run, and it passed.
That felt like assurance and was not. It validates that the file is well-formed
YAML in the expected shape. It does not check that the pinned revisions exist —
and four of five pins pointed at tags that had never been released.

| | |
|---|---|
| **Reads as** | "Config validated" |
| **Actually** | "This is valid YAML." Two different claims sharing a word. |

Semgrep also had no Windows build and could never have run locally at all, a
fact no amount of config validation would surface.

**The check:** for every passing check, name the property it actually verified.
Not "it's fine" — the specific claim. *Valid syntax. Tags resolve. Tool
executes on this platform. Tool found files to look at. Tool found a problem
and failed.* Those are five different guarantees, and tools routinely give you
the first while you assume the fifth.

---

## 5. The trigger gap

The only one needing two files open at once, and the one that produced a signed
artifact asserting something untrue.

```yaml
# ci.yml — everything that gates quality
on:
  push:
    branches: [main]
  pull_request:
  schedule: [{ cron: "0 6 * * 1" }]

# release.yml — publishes a release with an SBOM attached
on:
  push:
    tags: ["v*"]
```

| | |
|---|---|
| **Reads as** | CI gates everything; releases are cut from gated code |
| **Actually** | A tag is not a push to `main`. Tagging runs the release workflow *alone*. |

With no branch protection, a direct push of a red `main` plus a tag produced an
official release, SBOM attached, advertising supply-chain evidence for a build
whose vulnerability gate may never have run. An ancestry check did not catch it
either: every commit in `main`'s history is an ancestor of `main`, so the test
was true and meaningless.

**The check:** wherever one workflow's promise depends on another having run,
put their `on:` blocks side by side. **The gap between them is where the
guarantee dies.**

---

## What honest looks like

The same step, failing closed. The exemption is not removed — it is *named*,
scoped and loud.

```yaml
run: |
  if [ -z "$TS_API_KEY" ]; then
    # The one legitimate exemption. Dependabot PRs run with their own
    # secret store and cannot read Actions secrets at all, so failing
    # them here would turn every dependency PR red for a reason the
    # author cannot fix. Narrow, and named rather than silent.
    if [ "${{ github.actor }}" = "dependabot[bot]" ]; then
      echo "::notice::Dependabot PR — no access to Actions secrets."
      exit 0
    fi
    echo "::error::TS_API_KEY is not set, so the tailnet ACL was not validated."
    exit 1
  fi
```

Three properties worth stealing:

- **Default.** A missing precondition means **red**. The build going red when
  the key is absent *is the signal*, not a defect to work around.
- **Exemption.** One case, justified in a comment explaining why that PR's
  author could not fix it. Anyone can audit whether it is still true.
- **Volume.** `::error::`, not `::warning::`.

The release fix has the same shape: require at least one successful CI run for
that exact SHA, and grant `actions: read` — because a permissions error would
make the check itself throw, and **a check that errors is a check that does not
protect.**

---

## Handed an unfamiliar pipeline

1. **Did it run at all?** Recent runs, on this branch. A workflow with no runs
   is a document.
2. **How long did each job take?** Anything far under its honest floor did not
   do the work.
3. **Grep for the off switches.** `exit 0`, `|| true`, `continue-on-error`.
4. **Name what each green check verified.** Syntax, resolution, execution,
   coverage, detection — five different claims.
5. **Diff the triggers** of workflows that depend on each other.
6. **Break it on purpose, once.** Commit a fake secret. Push a lint error. If
   the build does not go red, you have learned the only thing that matters —
   and it is the step that would have caught all five of the above on day one.

The lesson from the incident, written before most of the rest was found:
**a control that has never executed is not a control. It is a description of
one.**
