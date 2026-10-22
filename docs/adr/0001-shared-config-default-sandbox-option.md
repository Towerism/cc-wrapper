# ADR 0001: Local mode shares the normal config; the sandbox is an option

**Status:** proposed, 2026-10-22. It becomes accepted when the owner approves it.

`ccx local` runs `claude` with the normal config directory, so skills, memory, permissions and `/resume` history are shared with plain `ccx`. `ccx local --sandbox` uses a separate directory for experiments with the config itself. This is a prototype decision and can change (see [ADR 0007](0007-prototype-status-and-unstable-interface.md)).

## Context

The goal is to switch between Claude and a local model without keeping two copies of projects, history or config. `claude` takes its config directory from `CLAUDE_CONFIG_DIR` and falls back to the standard location in the home directory. The backend comes from other environment variables, so the two choices are independent: any mode can use either directory.

Observed with Claude Code 2.1.299 (see [the evidence record](../evidence/resume-across-backends.md)): one config directory and project resumed under two different backend environments. The headless `--resume` and the interactive `/resume` picker both worked, and the resumed request carried the earlier turns. Also observed: with `ANTHROPIC_AUTH_TOKEN` set and a real login in the config directory, `claude` sent the configured token and no stored login to the endpoint.

## Options

| Option | For | Against |
|---|---|---|
| A. Shared config by default, sandbox on request | No duplicate setup. History and memory carry across. One flag for experiments. | Permission allow-lists and skills written for Claude apply to a weaker model. Hooks run in both modes. |
| B. Separate config always | Local mode cannot touch the real config. | Two copies of everything. `/resume` cannot cross modes. Defeats the purpose. |
| C. Shared config only, no sandbox | Simplest. | Changing the config means changing the one real copy. |

## Decision

Option A. The sandbox is only under `local`, and `ccx` sets both `CLAUDE_CONFIG_DIR` and `CLAUDE_HOME` for it, because the `Towerism/claude-config` hooks read `CLAUDE_HOME` and fall back to the real directory when it is unset.

## Consequences

- Sessions started in one mode can be resumed in the other, verified for two fake backends only. Plain mode with real credentials is unverified.
- Sandbox isolation is incomplete. The deployed global `CLAUDE.md` names the real `~/.claude/skill-overlays/...` path as a fallback, so the sandbox can still read it. Skills added by hand or synced from claude.ai are not copied by `cfg deploy --target <dir>`, which manages only the paths in its host profile.
- Settings in the shared config that set a model or environment variables could fight the variables `ccx` sets. This was not tested and is an open question.
- Permission rules accepted for Claude are accepted for the local model. Users who want stricter rules for the local model have to use the sandbox or per-run flags.

## Revisit when

A settings conflict is found; the resume check fails against real servers; or a second kind of isolation, such as a read-only copy, is wanted.
