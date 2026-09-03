# ADR-0006: Actions that spend money require explicit confirmation

- **Status:** Accepted
- **Date:** 2026-08-10
- **Deciders:** Jon Hauk

## Context

`genai-studio` exposes local generation (free) and cloud generation (billed per
second) behind one `Provider` protocol. That is the point of the abstraction —
app code should not care which backend it got.

It is also the risk. Cloud video is roughly $0.08/second. A retry loop, a test
suite in watch mode, or a `for` loop over prompts can spend real money at
machine speed, and the abstraction is specifically designed to make the
expensive path look identical to the free one.

The failure here is quiet and cumulative: nothing errors, nothing alerts, and
you find out on a statement.

## Decision

Any provider that spends money must:

1. **Never be invoked implicitly.** No default, no fallback, no automatic
   retry into a paid path.
2. **Estimate before spending**, and require an interactive confirmation
   showing that estimate — unless the caller passes an explicit `--yes`.
3. **Report actual cost** on the returned `GenerationResult`.

Free providers return `cost_usd = 0.0`, not `None`, so callers can distinguish
"this is free" from "cost unknown". Local image generation returns `None`
because electricity is not itemised.

The default video provider is `local`. Reaching fal requires typing
`--provider fal`.

## Alternatives considered

**Option A — a global spend cap.** Genuinely good, and worth adding later.
Rejected as the *primary* control because it is a backstop, not a gate: it
limits the damage of an accident without preventing it, and a cap large enough
to be useful is large enough to hurt.

**Option B — an env var like `GENAI_ALLOW_PAID=1`.** Rejected: set once in a
shell profile and it is permanently on, which is the same as no gate. Worse, it
is invisible at the call site.

**Option C — trust the caller.** Rejected. The whole design goal of the
provider protocol is that callers cannot tell the backends apart.

## Consequences

**Good**
- No path from a typo or a loop to an unbounded bill.
- The cost is visible at the moment of decision, not afterwards.
- Tests can exercise the entire paid code path with the network mocked, so CI
  costs nothing. The one genuinely billed test is marked `paid` and excluded
  from the default run.

**Bad / accepted costs**
- Interactive confirmation is hostile to automation. `--yes` exists for that,
  which is a deliberate escape hatch and therefore a deliberate risk.
- The rate table in `fal_video.py` is hardcoded and will drift from fal's
  actual pricing. It is a warning, not an invoice — documented as such.

**Neutral**
- One extra keystroke per paid generation.

## What would change this decision

If a spend cap and billing alerts exist at the account level, the interactive
confirmation could reasonably relax to a first-use-per-session prompt.

## References

- `genai/src/genai_studio/providers/fal_video.py`
- `genai/src/genai_studio/cli.py` — `video` command
- `genai/tests/test_video_mocked.py` — paid path, zero cost
- `genai/tests/test_live_paid.py` — the one billed test
