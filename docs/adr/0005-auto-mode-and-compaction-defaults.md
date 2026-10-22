# ADR 0005: Local mode does not use auto permission mode; compaction and output limits come from config

**Status:** proposed, 2026-10-22. It becomes accepted when the owner approves it.

`ccx local` passes `--permission-mode default` unless the user sets a mode. The compaction window and the maximum output tokens come from the user's config, and `ccx` warns when they are missing.

## Context

Observed with Claude Code 2.1.299 and a throwaway listener (see [the evidence record](../evidence/resume-across-backends.md)):

- At launch against a custom endpoint, the interactive status line showed auto mode on. With `--permission-mode default` it did not.
- A notice said that auto mode's classifier requests go through the same endpoint as the session. In auto mode, the model the user points `claude` at is therefore also asked to decide which actions are safe. A weaker local model makes that decision less reliable, and the decision guards file and shell actions.
- The first request had no user words in it yet and was about 87,000 characters, roughly 20,000 tokens at four characters per token (an estimate), dominated by 21 tool definitions.
- `claude` asked for 32000 output tokens by default. With `CLAUDE_CODE_MAX_OUTPUT_TOKENS` set to 4096, it asked for 4096.

A local model's context is much smaller than Claude's, and `ccx` cannot know how large it is: the size depends on the server and its settings, which `ccx` does not assume ([ADR 0003](0003-configuration-over-hardcoded-server-details.md)).

## Options

Permission mode:

| Option | For | Against |
|---|---|---|
| A. Pass `default` unless configured | Every action is asked or follows the user's rules, and a weak model does not judge safety | More prompts than auto mode |
| B. Leave `claude`'s default | No `ccx` opinion | Observed to be auto mode here, which hands safety checks to the local model |
| C. Always auto mode | Fewest prompts | The same safety concern, chosen on purpose |

Compaction and output limits:

| Option | For | Against |
|---|---|---|
| 1. Config values, no defaults, a warning when absent | Matches the server the user has | A user who ignores the warning risks overflow |
| 2. Built-in defaults | Works at once | Any number is a guess about someone else's server |

## Decision

Permission mode: option A. `local.permission_mode` overrides it, and a `--permission-mode` or `--dangerously-skip-permissions` argument on the command line wins over both. Plain `ccx` never changes the mode.

Limits: option 1. `local.compact_window` sets `CLAUDE_CODE_AUTO_COMPACT_WINDOW` and `local.max_output_tokens` sets `CLAUDE_CODE_MAX_OUTPUT_TOKENS`. When either is missing, `ccx` does not set the variable and prints one line on launch naming the key. The documentation gives guidance, not numbers: set the compaction window below the server's context length, leaving room for the output limit and for the roughly 20,000 tokens that the tool definitions and system text already take.

## Consequences

- Local sessions prompt more often than they would in auto mode. A user who accepts the risk can set `local.permission_mode`.
- The warning is shown on every launch until the keys are set. That is intended, and a user can silence it only by setting them.
- The variable names for compaction, telemetry and updates were taken from the earlier prototype. Only `CLAUDE_CODE_MAX_OUTPUT_TOKENS` was confirmed against this version.
- If the server does not support the classifier requests, a user who turns auto mode on may see failures. That was not tested.

## Revisit when

A newer Claude Code changes its permission-mode default; auto mode proves reliable with local models; or the compaction variable is renamed.
