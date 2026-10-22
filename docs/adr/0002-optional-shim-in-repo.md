# ADR 0002: The shim ships in this repo, is optional, and starts on demand

**Status:** proposed, 2026-10-22. It becomes accepted when the owner approves it. The location is already an owner decision: on 2026-10-22 the owner said "the shim does ship in this repo".

The interface is unstable ([ADR 0007](0007-prototype-status-and-unstable-interface.md)); undecided items are in [open-questions.md](../open-questions.md).

The shim is the `ccx shim` subcommand of the single `ccx` file. `ccx local` starts one shim per session when it is needed, on a loopback address with an operating-system-chosen port, and stops it when `claude` exits.

## Context

Claude Code can send a `role: "system"` message after the first position of `messages`. Some chat templates reject that with `System message must be at the beginning`, and the first request fails with a 500. A small pass-through proxy fixed it in the earlier attempt: it turned non-leading `system` messages into `user` turns, merged adjacent user turns, forwarded everything else and streamed the response.

Evidence, from the earlier attempt: the failure with a Qwen chat template and a hook that adds context. Evidence from this design pass, Claude Code 2.1.299 only: an empty config directory with no hooks still sent the roles `user, system`, and a resume under a new model added another `system` message. So the need does not depend on hooks (see [the evidence record](../evidence/resume-across-backends.md)). Whether the server rejects the message depends on its chat template, which `ccx` must not assume.

The earlier prototype hardcoded a default listening address for its shim. That is the kind of assumption [ADR 0003](0003-configuration-over-hardcoded-server-details.md) rules out.

## Evidence

- The failure with a Qwen chat template and a hook that adds context is from the earlier attempt and was not reproduced here.
- In this design pass, Claude Code 2.1.299 sent a mid-array `system` message with an empty config directory and no hooks, and another after a cross-backend resume ([evidence record](../evidence/resume-across-backends.md)). One version only.
- The probe and the per-session start and stop design are untested.

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
| 2. A control and test probe, cached | Finds the need without user knowledge, whatever the error wording | Sends two extra inference requests |
| 3. Parse the first failure and retry | No extra request | Complicated: the failure happens inside `claude` |

## Decision

Start: option A. The shim reports its address to `ccx` on its standard output. It reads its standard input and exits on end of file, so a crashed `ccx` leaves no orphan. It runs in its own session so a terminal Ctrl-C does not kill it. The details are in [the architecture document](../architecture.md#process-model-with-the-shim).

Decide: `local.shim` is `auto`, `always` or `never`, default `auto`. In `auto`, `ccx` sends a control request and a test request (a `system` message after a `user` message), each limited to a few output tokens, and uses the shim only when the control is accepted and the test is rejected. Any other outcome means unknown: launch without the shim and warn, naming `local.shim = "always"`. The result is cached per base URL and model, with a time limit. The known error text only improves the message; it is not the test, because other templates fail with other words. Option 1 stays available through `always` and `never`.

When the shim is not used, `ccx` replaces itself with `claude`, so nothing of it stays running.

## Consequences

- The model sees hook and environment text as user-turn text, not system text. A model that gives system text more weight gives this less. Merged adjacent turns change the turn structure the model sees.
- The rewrite is deterministic, so prompt-cache prefixes stay stable.
- Text added by the conversion goes after `tool_result` blocks within a merged turn.
- `ccx` is a parent process during shim sessions and must forward signals and clean up on every exit path. This is a test target.
- The probe sends two real inference requests, costs tokens, can trigger a model load, and cannot see a server that silently drops the `system` text. A stored "no shim" can go stale when the server's template changes; the time limit, `local.shim = "always"` and a cache reset flag (name open) cover that.
- The shim needs to handle `https` upstreams, which the earlier prototype did not.

## Revisit when

A newer Claude Code stops sending non-leading `system` messages; a server accepts them; the probe proves unreliable on a real server; or a shared daemon becomes worth its cost.
