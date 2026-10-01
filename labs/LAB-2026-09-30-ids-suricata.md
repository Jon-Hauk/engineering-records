# LAB-2026-09-30: One Suricata alert, explained end to end

- **Goal:** reproduce one network signature live, show a benign request that
  does not trigger it, and prove the trigger payload at the packet layer.
- **Sensor:** Suricata in a pinned container on a Jetson Orin Nano, watching
  that host's own Wi-Fi traffic. Free, local, no cloud.
- **Result:** PASS, 2026-09-30. Independent technical and disclosure review
  passed.
- **Author:** Jon Hauk. I consulted Veridian for troubleshooting and
  explanation; Claude and Codex assisted with implementation and testing.

## What was tested

| Step | Expected | Observed |
|---|---|---|
| Rules loaded at start | 52,995 loaded, 0 failed | 52,995 loaded |
| Baseline | 0 alerts for the signature | 0 |
| Trigger 1 (`http://testmyids.com`) | count 1 | 1, at 14:11:28Z |
| Trigger 2 | count 2 | 2, at 14:11:59Z |
| Negative control (`http://example.com`) | HTTP event seen, count stays 2 | Seen at 14:12:38Z; count stayed 2 |
| Trigger 3, separate, under packet capture | count 3; payload present in the pcap | 3, at 14:14:24Z; payload present once |

The packet capture streams to stdout so the operator's shell owns the file,
not root. It is kept privately; only its SHA-256 is published with the lab.

## The signature

SID 2100498, `GPL ATTACK_RESPONSE id check returned root`, matches the bytes
`uid=0(root)` anywhere in IP traffic. testmyids.com returns that string
deliberately, so each request produced one alert.

A false positive is any traffic that merely contains that text, such as a web
page or a log quoting `id` output. That is the central lesson: **an alert is a
detected pattern, not proof of an intrusion.** This run demonstrates one
controlled detection on one host. It is not whole-network coverage, a
false-positive rate, or a SIEM.

## What went wrong on the way

**The sensor that could not alert (closed 2026-09-28).** The pinned image
ships with no rules. Packets flowed, nothing could fire, and nothing said so
loudly. The fix was a persistent rules volume plus a one-off `suricata-update`
(ET Open, fetched 2026-09-28 and frozen by hash). The run now stops at step
3 unless 52,995 rules load with zero failures. The startup log keeps the
0-rule start and the 52,995-rule start side by side.

**The check that failed while the sensor was healthy (2026-09-29).**
- *Symptom:* counting the rule in the rules file returned `Permission
  denied`.
- *Evidence:* the rules directory was mode 750, owned by the sensor's service
  user, and the startup log had already shown every rule loaded.
- *Root cause:* the checking command lacked traversal permission. The rules
  were never missing.
- *Fix:* the check now runs as the sensor's user in a read-only, no-network
  container with a read-only mount. No host permission was changed.
- *Prevention:* the check distinguishes "cannot read" from "rule absent".

**The capture that would have left root-owned files.** The inherited recipe
bind-mounted a host directory into a root-running capture container. It was
blocked before use and replaced by the stdout capture above, tested the day
before the run.

## Disclosure

Raw logs, the pcap and the 45.8 MB ruleset stay private. What is published is
allowlisted fields only, hashed. A redaction gate fails the commit on any
private, tailnet or link-local/unique-local address, MAC, or email address
(escaped forms included), and on binary, oversized or symlinked files.
Passing the gate is not publication clearance; human review still applies.

## Next

A Wazuh agent on the sensor shipping these alerts to a manager in a VM: the
SIEM half, so the alert becomes a decoded event someone can investigate.
