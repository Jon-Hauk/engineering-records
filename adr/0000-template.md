# ADR-0000: Short noun phrase describing the decision

- **Status:** Proposed | Accepted | Superseded by [ADR-00XX](00XX-name.md)
- **Date:** YYYY-MM-DD
- **Deciders:** who actually made the call

## Context

What is true that forces a decision? Constraints, threat model, the thing that
prompted this. Facts, not preferences — a reader should be able to disagree
with your conclusion while accepting this section.

State constraints explicitly, especially the boring ones. "12 GB VRAM" and
"solo operator, no second reviewer" are decision-shaping facts.

## Decision

What you are doing, in one or two sentences, active voice.

> We will ...

## Alternatives considered

The section that makes an ADR worth writing. Anyone can record what they did;
the value is in what you rejected and why, because that is what stops the
next person re-litigating it.

**Option A — <name>.** What it is. Why rejected.

**Option B — <name>.** What it is. Why rejected.

## Consequences

Honest, both directions.

**Good**
- ...

**Bad / accepted costs**
- ...

**Neutral**
- ...

## What would change this decision

The most useful section, and the one most people omit. Write the trigger that
should make a future reader reopen this.

> If X becomes true, revisit — the reasoning above assumes X is false.

## References

Links to the code this governs, docs, prior art.

---

## Operator

One line is enough. An unsigned record is indistinguishable from a draft.

- **Read:** YYYY-MM-DD — *(your initials or name)*
- **Decision:** `passed to reviewer` | `holding` | `override` | `rejected`
- **Notes:**

If **override**, these are required and not optional:

- **Risk accepted:** what specifically you are accepting
- **Reason:**
- **Expires:** YYYY-MM-DD *(7–14 days default; the override lapses, not the risk)*
- **Linked INC:**

---

## Reviewer sign-off

*(to be completed by the reviewer — approval status, findings, required hardening)*

- **Findings:**
- **Required hardening:**
- **Status:** Draft / Approved / Approved with Conditions / Rejected
- **Reviewed:**
