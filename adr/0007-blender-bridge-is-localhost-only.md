# ADR-0007: The Blender bridge binds localhost only, with a token, off by default

- **Status:** Accepted
- **Date:** 2026-08-10
- **Deciders:** Jon Hauk

## Context

The Blender addon accepts commands over HTTP so an external tool can drive
scene construction. One of those commands is `exec`, which runs arbitrary
Python inside Blender.

That is not a feature with a security caveat. **It is a remote code execution
endpoint**, deliberately, because arbitrary scripting is the entire point —
constraining it to a fixed command set would make it useless for ideation.

Blender's Python has full filesystem access and runs as the desktop user. A
request to this endpoint is equivalent to a shell.

There is a second, non-obvious hazard: `bpy` is not thread-safe. Touching it
from an HTTP handler thread does not raise — it **crashes Blender outright**,
losing unsaved work.

## Decision

Three constraints, none of which is optional:

1. **Bind `127.0.0.1` only.** Never `0.0.0.0`. Not reachable from the LAN or
   the tailnet.
2. **Require a bearer token**, generated fresh at start and written to
   `~/.genai-bridge-token`. Compared with `secrets.compare_digest`.
3. **Off by default.** The server starts only when explicitly enabled from the
   sidebar panel, and the token file is deleted on stop.

Remote access, when needed, is a Tailscale SSH tunnel — not a rebind:

```bash
ssh -L 8765:127.0.0.1:8765 <workstation>
```

Separately, all `bpy` work is marshalled onto Blender's main thread through a
queue drained by `bpy.app.timers`. The HTTP thread only enqueues.

## Alternatives considered

**Option A — bind `0.0.0.0` and rely on the token.** Rejected. It puts an RCE
endpoint on every network the laptop joins, including hotel wifi, defended by a
single secret in a plaintext file. The token is defence in depth, not a
perimeter.

**Option B — bind to the tailnet address.** Better than `0.0.0.0`, and
consistent with the ZTA design. Still rejected: it widens the blast radius from
"this machine" to "anything that reaches this machine over the tailnet",
including a compromised phone. Tunnelling gets the same access with strictly
less exposure, and costs one extra command.

**Option C — drop `exec` and expose only whitelisted operations.** Removes the
RCE surface entirely. Rejected because it removes the capability the bridge
exists for. The honest position is to keep the power and constrain the
reachability, not to pretend a scripting bridge is not a scripting bridge.

## Consequences

**Good**
- Not reachable from any network, so network position alone never grants access.
- The token rotates every start; a stale copy is useless.
- Off by default means the surface does not exist unless deliberately opened.
- The main-thread queue prevents a class of hard crash that costs unsaved work.

**Bad / accepted costs**
- Remote use requires setting up a tunnel.
- A token in a home-directory file is readable by anything already running as
  this user. Accepted: at that point the attacker can run Blender directly.
- Someone will eventually want to "just test it from the laptop" and rebind.
  That is why the reasoning is in the addon docstring as well as here.

**Neutral**
- Localhost HTTP is unencrypted. Irrelevant — it never leaves the machine.

## What would change this decision

If the bridge were ever used from a second machine routinely, the right answer
is still a tunnel, plus per-client tokens and an audit log of executed code —
not a wider bind.

## References

- `blender-ideation/addon/genai_bridge.py`
- ADR-0002 — why tailnet reachability is not the same as safe. Held back
  with the lab; see the numbering note in this directory's README.
