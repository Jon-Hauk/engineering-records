# pipeline

Reference copies, not a live workflow.

`ci.yml` is deliberately **not** under `.github/workflows/` in this repo. It
targets a private monorepo — it syncs a `uv` project in `genai/` and validates
a Tailscale ACL in `zta-soho/`, neither of which exists here. Dropped into
`.github/workflows/` it would fail on every push, and a portfolio repo with a
red badge argues against itself.

It is here to be read.

## What it runs

| Job | Purpose |
|---|---|
| `pre-commit` | Runs every hook against the whole repo. Exists because hooks were configured and then never executed — see [INC-2026-08-11](../incidents/INC-2026-08-11-devsecops-gates-never-executed.md). |
| `lint-and-test` | `ruff check`, `ruff format --check`, `mypy`, `pytest`, under `uv`. |
| `secrets` | gitleaks with `fetch-depth: 0`, so it scans history rather than the diff. |
| `sast` | semgrep against `p/python`, `p/secrets`, `p/github-actions`. |
| `dependencies` | Trivy filesystem scan, CRITICAL/HIGH fail the build, SARIF uploaded to code scanning. |
| `sbom` | CycloneDX SBOM, retained 90 days as a run artifact. |
| `zta-policy` | Validates the network policy against the Tailscale API. Skips with a warning when no key is present, rather than passing silently. |

## Notes worth stealing

**`permissions: contents: read` at the top level**, widened per job only where
needed — `security-events: write` appears once, on the job that uploads SARIF.

**The SBOM is retained, not just generated.** An SBOM you cannot retrieve
after the fact is not an SBOM.

**The weekly `schedule` trigger exists** because dependencies rot even when you
do not touch them. A CVE published against a pinned version you have not
changed is invisible to a push-triggered pipeline.

**`SKIP: psscriptanalyzer` in CI** — that hook needs Windows PowerShell and is
enforced on the developer machine instead. Recorded rather than silently
dropped.
