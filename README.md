# engineering-records

How I decide, and how I write up my own failures.

Three things live here: the architecture decision records for a private
security lab, the incident reports for controls that broke, and the CI
pipeline that enforces the gates those records argue for.

`pipeline/ci.yml` is a **reference copy, not a live workflow here** -- it
targets the private monorepo and would fail on every push in this repo.
[`pipeline/`](pipeline/) explains why that is deliberate.

The lab itself is private — it publishes the shape of a live network. These
are the parts that stand on their own.

## Why an incident report for a home lab

Because the interesting failure mode is not "the control was missing." It is
**"the control was written, documented as working, and had never once run."**
That happened here twice in one day, and
[INC-2026-08-11](incidents/INC-2026-08-11-devsecops-gates-never-executed.md)
is the write-up: four pre-commit hooks pinned to tags that did not exist, in a
repo where a commit with no hooks installed looks exactly like a commit where
every hook passed.

The rule that came out of it — execute a control once before calling it done —
is now the reason later ADRs carry per-claim evidence markers.

## Contents

| | |
|---|---|
| [`adr/`](adr/) | Architecture decision records. Why the security choices are what they are. |
| [`incidents/`](incidents/) | What broke, why, and what changed as a result. |
| [`pipeline/`](pipeline/) | The CI workflow and pre-commit config those decisions produced. Reference copies; they do not run in this repo. |

## Two rings

| | Pre-commit (local) | CI (GitHub) |
|---|---|---|
| **When** | `git commit` | push / PR / weekly |
| **Speed** | < 5 s, changed files | minutes, whole repo |
| **Skippable** | yes, `--no-verify` | no |
| **Purpose** | fast feedback | enforcement |

Both rings describe the monorepo they were built for, not this repository.

The local ring is deliberately bypassable. Making it unskippable just teaches
people to disable it entirely. CI is where the gate actually holds — which is
also why CI runs the pre-commit hooks itself, as a positive signal that the
hook config is not broken.

## What each tool earns its slot for

| Tool | Catches | Why |
|---|---|---|
| **gitleaks** | Committed secrets | A leaked key is the most common way a hobby repo becomes an incident. Full history in CI, not just the diff. |
| **semgrep** | Injection, unsafe deserialisation, subprocess misuse | Bug classes a formatter never finds. |
| **trivy** | CVEs in dependencies, IaC misconfig | Most of the risk is in code you did not write. |
| **syft** | SBOM (CycloneDX) | Lets you answer "were we exposed to X?" *after* X is disclosed. |
| **ruff** | Lint + format | Cheap, instant, removes style from review. |
| **mypy** | Type errors | `strict = true`. |
| **PSScriptAnalyzer** | PowerShell defects | The lab's scripts are real code and get real linting. |
| **ACL validation** | Broken network policy | A change that would let a phone SSH into a server fails the build. |

## License

[CC BY 4.0](LICENSE) — Creative Commons Attribution 4.0 International.

This repo is writing, not software, so a code licence fits it badly: Apache and
MIT are built around distributing and modifying source, and say nothing useful
about quoting a decision record. CC BY is the normal choice for documents. It
means anyone may share or adapt this, including commercially, provided they
credit it.

The code repos it describes are Apache-2.0 instead: `fieldkit`, `localagent`,
`win-sec-snapshot`.
