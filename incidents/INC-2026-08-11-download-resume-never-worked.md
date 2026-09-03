# INC-2026-08-11: Large model downloads never resumed, and the recovery made it worse

- **Severity:** SEV4 — no security impact; ~15 GB of bandwidth and several
  hours burned on a wrong model of the system
- **Detected:** 2026-08-11 17:50, by reading the `huggingface_hub` source
  instead of trusting its documented behaviour
- **Resolved:** 2026-08-11 17:57
- **Duration:** ~21 hours of intermittent failure across two sessions
- **Author:** Jon Hauk

## Summary

Multi-GB model downloads stalled repeatedly. The recovery strategy — kill the
stalled process, let a supervisor restart it, resume from the partial — was
built on an assumption that was false: `huggingface_hub` 1.27 writes each
download to a **process-unique** temporary file and cannot resume across
process restarts.

Every "fix" therefore destroyed the progress it was meant to preserve. A
13.6 GB file accumulated four separate partials totalling 5.4 GB and finished
none of them. Replaced with a downloader that resumes against a stable
filename via HTTP Range.

## Impact

- Roughly 15 GB of redundant download traffic.
- The Wan 2.2 i2v model failed to complete across two sessions.
- Several hours spent diagnosing "network throttling" that was partly
  self-inflicted.
- No security impact. No data loss beyond discarded partial files.

## Timeline

| Time | Event |
|---|---|
| 08-10 ~19:00 | First large download stalls. Diagnosed as HuggingFace throttling unauthenticated connections. Partly correct. |
| 08-10 ~19:15 | `HF_HUB_DOWNLOAD_TIMEOUT=20` added so a silent socket raises instead of hanging. Genuine improvement — stalls became visible errors. |
| 08-10 ~21:00 | Supervisor written to restart the fetcher on failure. Documented as "resumes from the partial, so a restart costs seconds, not GB." **This was asserted, never verified.** |
| 08-10 ~21:30 | An HF token is added. Throughput improves to 8-9 MB/s. Flux (16.4 GB) and Wan t2v (33 GB) complete overnight. The resume assumption is never tested, because nothing needed to resume. |
| 08-11 17:30 | i2v download repeatedly stalls. Each kill-and-restart appears to "start over"; noted as a suspicion in the day's notes but attributed to a possible bug in our own retry path. |
| 08-11 17:45 | Four `.incomplete` files observed for one target file, sharing a filename and etag but differing in a trailing 8-hex-character suffix. |
| 08-11 17:50 | Read `file_download.py`. Line 1948: `tmp_path = incomplete_path.with_name(f"{stem}.{uuid4().hex[:8]}.incomplete")`. The suffix is a fresh UUID per call. |
| 08-11 17:52 | Surrounding comment confirms it is deliberate (PR #4228): a shared `.incomplete` corrupts the cache where `flock` is unreliable, so "each process downloads the full file". |
| 08-11 17:55 | `genai_studio/downloads.py` written: stable `.part` file, `Range` resume, read timeout, in-process retry. |
| 08-11 17:56 | Largest surviving partial (3,005 MB) salvaged and renamed to the new stable path. |
| 08-11 17:57 | Restarted. Log reads `(resuming from 2.93 GB)`. Sustained 5.7 MB/s with a single growing `.part`. |

## Root cause

`huggingface_hub` 1.27 intentionally does not support cross-process resume.
Every call to `hf_hub_download` generates a new UUID-suffixed temporary file
and downloads from byte zero. `Range`/`resume_size` exist, but only for retries
**within** a single process.

The recovery architecture was built on the opposite assumption. Killing a hung
process was the only lever available to break a stall, and pulling that lever
discarded everything downloaded so far. The system was designed to defeat
itself, and it did so more effectively the more diligently it was operated.

## Why it took so long to see

- **The assumption was documented as fact.** "It resumes from the partial, so
  re-running costs nothing" was written into a commit message, a README, and a
  handoff doc. Repetition made it feel established. It was never once tested.
- **Partial progress masked total failure.** Each restart *did* download
  gigabytes. Watching the byte counter climb looked like progress toward
  completion rather than progress toward the same 30% four times.
- **A real confounder existed.** HuggingFace genuinely does throttle
  unauthenticated large downloads, and the token genuinely helped. Having one
  true cause made it easy to stop looking for a second.
- **The evidence was visible for hours and misread.** Multiple `.incomplete`
  files with different suffixes were printed in diagnostic output several
  times and read as "leftover junk from earlier attempts" rather than as the
  mechanism itself.

## What went well

- The `.part`/rename discipline in the replacement means a file that exists is
  always complete. There is no state where a truncated 13 GB file looks valid.
- 3,005 MB of the existing partial was salvageable, so the fix cost nothing
  extra to deploy.
- `HF_HUB_DOWNLOAD_TIMEOUT` was a real fix for a real problem and is retained
  in the new downloader as an explicit `read` timeout.

## Action items

| # | Action | Type | Status |
|---|---|---|---|
| 1 | Replace `hf_hub_download` with a stable-`.part` Range downloader for large ComfyUI files | prevent | done |
| 2 | Salvage the largest existing partial rather than discarding it | mitigate | done |
| 3 | Log `(resuming from N GB)` on every start, so resume is *observable* rather than assumed | **detect** | done |
| 4 | Log throughput periodically so a stall is visible in the log itself | **detect** | done |
| 5 | Add a test that a partial file is actually resumed rather than restarted | detect | done |
| 6 | Correct the false "resume" claims in commit history and README | prevent | done |

Item 3 is the one that matters. The bug was invisible because nothing ever
printed what the code was actually doing.

## Lessons

**An assumption written down repeatedly is still an assumption.** This claim
appeared in a commit message, a README, and a handoff document before anyone
checked it. Documentation confers no truth; it only spreads the belief faster.

**When a library's behaviour matters, read the library.** Thirty seconds of
`grep incomplete file_download.py` would have saved several hours. The answer
was not in the docs — it was in a source comment explaining a deliberate
trade-off made for a use case that is not ours.

**A recovery mechanism can be the failure.** The kill-and-restart loop was
built to make downloads robust and was the reason they never finished. When a
system fails more the harder you operate it, suspect the operating procedure
before the environment.

**Make the invisible visible before optimising it.** Three separate strategies
were tried against this problem — read timeouts, a watchdog supervisor,
sequential downloads — with no instrumentation showing whether resume worked.
One log line stating what the code was about to do would have exposed it
immediately.

## References

- `huggingface_hub/file_download.py:1948` and the comment above it
- [huggingface_hub PR #4228](https://github.com/huggingface/huggingface_hub/pull/4228)
- `genai/src/genai_studio/downloads.py`
