# ADR 0002: The shim ships in this repo, is optional, and starts on demand

**Status:** the location is an owner decision, 2026-10-22 ("the shim does ship in this repo"). The rest is proposed and becomes accepted when the owner approves it.

The shim is the `ccx shim` subcommand of the single `ccx` file. `ccx local` starts one shim per session when it is needed, on a loopback address with an operating-system-chosen port, and stops it when `claude` exits.

## Context

Claude Code can send a `role: "system"` message after the first position of `messages`. Some chat templates reject that with `System message must be at the beginning`, and the first request fails with a 500. A small pass-through proxy fixed it in the earlier attempt: it turned non-leading `system` messages into `user` turns, merged adjacent user turns, forwarded everything else and streamed the response.

Evidence, from the earlier attempt: the failure with a Qwen chat template and a hook that adds context. Evidence from this design pass, Claude Code 2.1.299 only: an empty config directory with no hooks still sent the roles `user, system`, and a resume under a new model added another `system` message. So the need does not depend on hooks (see [the evidence record](../evidence/resume-across-backends.md)). Whether the server rejects the message depends on its chat template, which `ccx` must not assume.

The earlier prototype hardcoded a default listening address for its shim. That is the kind of assumption [ADR 0003](0003-configuration-over-hardcoded-server-details.md) rules out.

## Options

How the shim is started:

| Option | For | Against |
|---|---|---|
| A. One shim per session, a child of `ccx` | No shared state, no stale process, no service manager, no fixed port | `ccx` stays running as a parent while the shim is used |
| B. A long-lived shared daemon | One process for many sessions | Needs a way to find it, a port convention or a state file, stale-process handling and a way to stop it |
| C. Always run the shim | Simplest logic | Adds a hop and a failure point when the server does not need it |

How `ccx` decides it is needed:

| Option | For | Against |
|---|---|---|
| 1. A config switch only | Trivial, predictable | The user must diagnose a 500 and set it |
| 2. A probe request, cached | Finds the need without user knowledge | Sends one extra inference request. Matches the known error text, the only server behaviour `ccx` assumes |
| 3. Parse the first failure and retry | No extra request | Complicated: the failure happens inside `claude` |

## Decision

Start: option A. The shim reports its address to `ccx` over a pipe. It exits when that pipe closes, so a crashed `ccx` leaves no orphan.

Decide: `local.shim` is `auto`, `always` or `never`, default `auto`. In `auto`, `ccx` sends one probe, a message list with a `system` message after a `user` message, with a one-token limit, and caches the result per base URL and model in its state directory. Accepted means no shim. Rejected with the known error text means shim. Any other outcome means unknown: launch without the shim and print a warning that names `local.shim = "always"`. Option 1 is always available through `always` and `never`.

When the shim is not used, `ccx` replaces itself with `claude`, so nothing of it stays running.

## Consequences

- The model sees hook and environment text as user-turn text, not system text. A model that gives system text more weight gives this less. Merged adjacent turns change the turn structure the model sees.
- The rewrite is deterministic, so prompt-cache prefixes stay stable.
- Text added by the conversion goes after `tool_result` blocks within a merged turn.
- `ccx` is a parent process during shim sessions and must forward signals and clean up on every exit path. This is a test target.
- The probe uses a real inference request and a stored result can go stale when the server's template changes. `local.shim = "always"` and a cache reset option cover that; the reset option's name is open.
- The shim needs to handle `https` upstreams, which the earlier prototype did not.

## Revisit when

A newer Claude Code stops sending non-leading `system` messages; a server accepts them; the probe proves unreliable on a real server; or a shared daemon becomes worth its cost.
