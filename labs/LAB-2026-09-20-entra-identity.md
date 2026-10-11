# LAB-2026-09-20: An Entra lab built as code, and the check that reads it

- **Goal:** stand a Microsoft Entra (Microsoft 365) lab up from nothing with
  known-bad state planted on purpose, read it back with a read-only posture
  check, tear it down, and prove nothing survived with an audit that does not
  trust the teardown's own report.
- **Platform:** a free-tier Entra tenant with one P2 trial seat. Windows
  PowerShell 5.1 and `Microsoft.Graph.Authentication` 2.39.0. Device-code
  sign-in only; no stored credentials.
- **Result:** PASS, 2026-09-20. Every step ran live and was checked by an
  independent read. One later re-run (2026-10-10) was refused by design; see
  below.
- **Author:** Jon Hauk, operator of every live run and sign-in. Codex built the
  lab and the check from a written brief; Claude audited each change (nine
  audits) and drafted this record from the private review log.

## What was built

| Piece | What it does |
|---|---|
| Engagement gate | No live run without a signed engagement record: schema, tenant ID, dates and pack version are checked before any sign-in. |
| Up | Idempotent. Declares its human steps, refuses any tenant but the one in the record, and plants three default objects it records in an ownership ledger: a privileged member (Global Administrator, password only), that role assignment, and an unredeemed guest invitation. |
| Read-only check | `fieldkit -M365`: six delegated read scopes, pseudonymised reports, raw output under a seven-day retention manifest. It refuses any session carrying a scope beyond those six. |
| Down | Deletes only ledger-owned objects, purges soft-deleted users, and finishes with a leftover audit. |
| Leftover audit | Runs separately as a read-only identity in a fresh process and inventories 13 resource types for anything the lab could have left behind. |

## What was tested

| Step | Expected | Observed |
|---|---|---|
| Up | Three planted objects, all in the ledger | 3 |
| Read-only check, seven resources | Each count and setting matches a direct Graph read made separately by the operator | 7 of 7 matched |
| MFA registration | Registration state for each enabled account | Unknown for 4 of 4: the registration report had not backfilled. Reported as `UNKN`, never as "no MFA". |
| Down | `Clean: True` | Clean |
| Leftover audit, read-only identity, fresh process | 13 of 13 `INFO`, zero lab candidates | 13 of 13, zero |
| CI on the final change | All required checks green | 16 of 16 |

## What went wrong on the way

All five of these passed every offline test and failed only against the live
tenant. A fixture cannot find what it does not model.

- **L1: the sign-in code never appeared.** The Graph SDK prints the
  device-code prompt on PowerShell's output stream, and `$null =` in front of
  `Connect-MgGraph` swallowed it. The process sat polling with nothing on
  screen. Reproduced with and without `$null =`; the fix pipes the prompt to
  the host.
- **L2: only one resource of seven was ever collected.** In SDK 2.39.0, any
  call to `Set-MgRequestContext` after a request has been sent makes every
  later Graph request fail with an "Invalid URI" error, for valid URIs. Found
  with a five-variant test in one signed-in session. The fix sets the context
  once per process, before the first request.
- **L3: teardown could never report clean.** Entra rewrites a soft-deleted
  user's sign-in name with a 32-character object-ID prefix so the name can be
  reused. The ownership check expected the original name, so every purge was
  refused as identity drift. The check now accepts exactly that prefixed form
  and still refuses any other drift.
- **L4: the leftover audit could never be clean either.** It reported Azure
  coverage as unknown whenever no Azure session existed, although the lab
  plants nothing in Azure. Coverage is now reported as "not assessed", with
  the claim scoped to the lab's own objects.
- **L5: the standalone audit failed before sign-in.** Its entry point
  imported one module, and the gates it called live in a module that one
  imports privately. Every unit test imported every module, so the suite was
  green. The fix adds the import plus a test that runs the real entry point in
  a fresh `powershell.exe` with no modules preloaded.

## 2026-10-10: the guard said no

A re-run to fill in the MFA figures was refused at session validation. The
read-only identity still held two extra read permissions granted earlier for
the leftover audit. A Graph CLI token carries every permission consented for
that user, and the check refuses anything beyond its six. That is the designed
behaviour: it stopped rather than run with more access than the engagement
approved.

It also exposed a design conflict, now recorded as open: the leftover audit and
the posture check cannot share one read-only identity while the tenant blocks
user consent. The options are a second identity, a separate app registration
per check, or a narrowly argued exception in the guard.

## Disclosure

The tenant ID, domain, account names, object IDs and raw output stay private.
Reports are pseudonymised by default. The read-only check is public in
[fieldkit](https://github.com/Jon-Hauk/fieldkit). The lab's provisioning code
and the full review log remain in the private monorepo.

## Next

The SIEM half of the IDS lab: Suricata alerts shipped to a Wazuh manager and
shown as decoded events.
