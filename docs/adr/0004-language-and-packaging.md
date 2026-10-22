# ADR 0004: TypeScript on Node.js, built into one file

**Status:** proposed, 2026-10-22. It becomes accepted when the owner approves it.

The interface is unstable ([ADR 0007](0007-prototype-status-and-unstable-interface.md)); undecided items are in [open-questions.md](../open-questions.md).

`ccx` is written in TypeScript for Node.js and built into one executable JavaScript file. Commander, the command-line parsing library, is its one runtime dependency and is bundled into the built file, so nothing has to be installed alongside it. Build and development tools are ordinary dependencies. The choice of TypeScript and of "Node, built bundle" is the owner's, made on 2026-10-22, and replaces an earlier proposal of one Python 3 standard-library file.

## Context

The owner's goals for this decision are readability, maintainability and simplicity, and an install that is easy. "No dependencies" is not a goal in itself. In the owner's words: "with typescript no-dependencies can be a goal for the shipped package, but dev dependencies can significantly increase DX". The owner then allowed one runtime dependency: "commander can be the one dependency".

`ccx` has to read a config file, make HTTP requests, run an HTTP proxy that streams (the shim), start a child process and pass signals, using the installed `claude` and nothing from the local server's own tooling. Node.js has these built in (`node:http`, `node:https`, `node:child_process`, `node:fs`, `process`). It has no parser for TOML (a plain-text configuration format), and JSON needs none.

## Evidence

None measured. This is the owner's decision. The earlier Python prototype showed that the work fits in a few hundred lines in either language; that comes from reading its source, not from a comparison.

## Options

| Option | For | Against |
|---|---|---|
| A. TypeScript on Node.js, built into one JavaScript file | Typed code that is readable and testable, split into modules in the repository. One file to install. The build target sets the Node versions it runs on. Build tools never reach the user. | A build step. Users need Node.js. |
| B. TypeScript run directly by a recent Node (types stripped, no build) | No build step | Only recent Node versions, and only syntax that can be erased, so no enums or parameter properties |
| C. One Python 3 standard-library file (the earlier proposal) | No build, no dependencies | Replaced by the owner's choice. Needs Python 3.11 for TOML. One file grows. |
| D. Bun compiled executable | No runtime needed on the user's machine | Needs Bun to build and a release for each platform |
| E. Deno | Runs TypeScript directly | The least common runtime, so the hardest install |
| F. Bash | Always present | A poor fit for the shim and config parsing. Hard to test. |
| G. Option A with no command-line library | No dependency at all | Hand-written parsing, help text and usage errors that Commander already provides |

## Decision

Option A. The owner chose "Node, built bundle".

- **Layout (proposed).** TypeScript sources under `src/`, split by job: argument parsing, config, environment building, the reachability check, the probe, the shim, and launching `claude`. A bundler (esbuild is proposed) builds them into one file, `dist/ccx`, that starts with a `#!/usr/bin/env node` line. `tsc --noEmit` type-checks.
- **Dependencies.** At run time the code uses Node's own modules and Commander, which parses the command line (subcommands, the `ccx` flags, `--dry-run` output and error messages for bad usage). esbuild bundles Commander into `dist/ccx`. Because Commander is the one runtime dependency, the config file is JSON, which Node reads with no library. A parser for TOML or for JSON with comments (JSONC) would be a second dependency, so it is not proposed. The owner did not name a format; this follows from the one-dependency decision and is easy to change (open question 27).
- **Tests (proposed).** A test runner and a pseudo-terminal driver are development dependencies. Vitest and a pty library are proposed. Which ones is decided when the tests are written.
- **Install and first-run steps,** the ease-of-use criterion for this decision:
  1. Get the built file. Either clone the repository, run `npm ci` and `npm run build`, or download a released `dist/ccx` if releases publish it (an open question).
  2. Put it on `PATH` as `ccx`, for example `install -m 755 dist/ccx <directory-on-PATH>`.
  3. Run `ccx init`. It writes `~/.config/ccx/config.json` (or under `XDG_CONFIG_HOME`) with placeholders, and refuses to overwrite an existing file.
  4. Edit three values: `local.base_url`, `local.model` and the credential source, `local.token_file` (or `local.token_env` in its place).
  5. Run `ccx local --dry-run` to see the environment and command line, with the credential shown as `<redacted>`.
  6. Run `ccx local`.

## Consequences

- `ccx` runs wherever a supported Node.js and `claude` are present. The lowest supported Node version is an open question; the bundle targets it.
- Users without Node.js cannot run it. Whether a compiled executable (option D) is worth offering is an open question.
- Source modules stay small, so the "one file grows" concern of the Python proposal applies only to the built file, which nobody edits.
- The built file contains Commander, so it must be rebuilt to pick up a Commander fix. The repository has a lockfile and development dependencies to keep current.
- Wherever these documents say "standard library", they now mean Node's built-in modules: the shim uses `node:http` and `node:https`, and process handling uses `process` and `node:child_process`.

## Revisit when

A Node version problem appears for users, users without Node ask for an executable, or the build step becomes a burden for a prototype.
