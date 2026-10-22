# ADR 0008: A TypeScript library API is a late milestone

**Status:** proposed, 2026-10-22. It becomes accepted when the owner approves it.

The interface is unstable ([ADR 0007](0007-prototype-status-and-unstable-interface.md)); undecided items are in [open-questions.md](../open-questions.md).

`ccx` will also be offered as a TypeScript library, so that a command-line application can use Claude Code with a local model as its harness, fully configured by an optional config directory. It comes late in the work, after the CLI, the shim, sandbox mode and the tests. The request is the owner's, made on 2026-10-22: "a typescript API would be good so that claude with a local model can be used as a harness in CLI applications. the harness can be fully configured with an optional config directory. this can be one of the last milestones."

## Context

The CLI already has to select a backend, check reachability, decide on the shim, build the environment and launch `claude`. An application that wants the same behaviour would have to copy it. [ADR 0004](0004-language-and-packaging.md) splits the TypeScript source by job, which makes sharing the modules natural.

## Evidence

None measured. This is the owner's requirement. The sandbox mode's use of `CLAUDE_CONFIG_DIR` and `CLAUDE_HOME` ([ADR 0001](0001-shared-config-default-sandbox-option.md)) is the existing mechanism for a config directory that is separate from the user's.

## Options

| Option | For | Against |
|---|---|---|
| A. A library over the same modules as the CLI, starting `claude` as a headless child | One implementation. Same version-support goal as the CLI. | The surface depends on `claude`'s headless output format |
| B. A library built on a Claude agent SDK | Typed events from a maintained package | A runtime dependency and a second path to the backend, with its own version coupling |
| C. No library; applications call the `ccx` command | Nothing new to build | Applications parse command output and cannot manage the shim's lifetime |

## Decision

Option A as the working direction, with the choice between A and B left open until a spike compares them (open question 22). The API is a late milestone, its surface is a sketch, and an optional config directory fully configures it. If the directory is omitted, the harness behaves like `ccx local`. The details are in [the architecture document](../architecture.md#library-api-a-late-milestone).

## Consequences

- A library has callers, which sit badly with [ADR 0007](0007-prototype-status-and-unstable-interface.md)'s promise of nothing stable. At this milestone a stable surface, or a clear instability warning in the package, is needed (open question 25).
- Publishing a package becomes a real question: npm, module formats and type declarations (open question 26).
- The headless permission problem has to be solved, because nobody answers prompts (open question 24).
- The earlier milestones stay free of API concerns, except that their modules must not assume a terminal or a single launch per process.

## Revisit when

Another application needs the harness sooner than planned; the spike shows an agent SDK is the better base; or the headless output format of `claude` changes.
