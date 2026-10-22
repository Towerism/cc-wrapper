# ADR 0005: Local mode does not use auto permission mode; compaction and output limits come from config

**Status:** proposed, 2026-10-22. It becomes accepted when the owner approves it.

The interface is unstable ([ADR 0007](0007-prototype-status-and-unstable-interface.md)); undecided items are in [open-questions.md](../open-questions.md).

`ccx local` passes `--permission-mode default` unless the user sets a mode. The compaction window and the maximum output tokens come from the user's config, and `ccx` warns when they are missing.

## Context

Claude Code's own defaults matter here: it may start in auto mode against a custom endpoint, its first request is large, and it asks for a large output limit. The measurements are under Evidence.

A local model's context is much smaller than Claude's, and `ccx` cannot know how large it is: the size depends on the server and its settings, which `ccx` does not assume ([ADR 0003](0003-configuration-over-hardcoded-server-details.md)).

## Evidence

- Auto mode showed as on at launch once, in an interactive run whose environment held other inherited `CLAUDE_CODE_*` variables, so the source of that default is not isolated. With `--permission-mode default` it did not show ([evidence record](../evidence/resume-across-backends.md)).
- The first request of a headless run with a two-letter prompt was about 87,000 characters, roughly 20,000 tokens at four characters per token (an estimate), mostly 21 tool definitions.
- `claude` asked for 32000 output tokens by default, and 4096 when `CLAUDE_CODE_MAX_OUTPUT_TOKENS` was 4096.

## Options

Permission mode:

| Option | For | Against |
|---|---|---|
| A. Pass `default` unless configured | Every action is asked or follows the user's rules, and a weak model does not judge safety | More prompts than auto mode |
| B. Leave `claude`'s default | No `ccx` opinion | Seemed to be auto mode in one contaminated run; if so, it hands safety checks to the local model |
| C. Always auto mode | Fewest prompts | The same safety concern, chosen on purpose |

Compaction and output limits:

| Option | For | Against |
|---|---|---|
| 1. Config values, no defaults, a warning when absent | Matches the server the user has | A user who ignores the warning risks overflow |
| 2. Built-in defaults | Works at once | Any number is a guess about someone else's server |

## Decision

Permission mode: option A. `local.permission_mode` overrides it. `ccx` adds `--permission-mode <mode>` only when the first `claude` argument is absent or starts with `-` (so `ccx local mcp list` or `ccx local update` is untouched, because an option before a subcommand would break it) and only when no `--permission-mode`, `--permission-mode=<mode>` or `--dangerously-skip-permissions` appears before a bare `--`. That is the one place `ccx` reads `claude` arguments. Plain `ccx` never changes the mode.

Limits: option 1. `local.compact_window` sets `CLAUDE_CODE_AUTO_COMPACT_WINDOW` and `local.max_output_tokens` sets `CLAUDE_CODE_MAX_OUTPUT_TOKENS`. When either is missing, `ccx` does not set the variable and prints one line on launch naming the key. The documentation gives guidance, not numbers: set the compaction window below the server's context length, leaving room for the output limit and for the roughly 20,000 tokens that the tool definitions and system text already take.

## Consequences

- Local sessions prompt more often than they would in auto mode. A user who accepts the risk can set `local.permission_mode`.
- The warning is shown on every launch until the keys are set. That is intended, and a user can silence it only by setting them.
- The variable names for compaction, telemetry and updates were taken from the earlier prototype. Only `CLAUDE_CODE_MAX_OUTPUT_TOKENS` was confirmed against this version.
- If the server does not support the classifier requests, a user who turns auto mode on may see failures. That was not tested.

## Revisit when

A newer Claude Code changes its permission-mode default; auto mode proves reliable with local models; or the compaction variable is renamed.
