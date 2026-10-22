# ADR 0004: One Python 3 standard-library file, installed by copying it

**Status:** proposed, 2026-10-22. It becomes accepted when the owner approves it.

`ccx` is a single executable Python 3 file that uses only the standard library. Installing it means putting that file on `PATH`. It depends on `claude` and on nothing from the local server's own tooling.

## Context

Requirements: work standalone with plain `claude` and bash or the Python 3 standard library; be easy to install and configure; include an HTTP proxy that streams (the shim); read a config file; be testable. The owner's machine already has a Python 3 that is newer than 3.11.

## Options

| Option | For | Against |
|---|---|---|
| A. Python 3 standard library, one file | No dependencies. Handles HTTP streaming, TOML (a plain-text configuration format) reading in Python 3.11 and later and subprocesses. Install is a copy. The earlier prototype was this shape. | Needs Python 3.11 or later. One file grows; it is split when it gets hard to read. |
| B. Bash | Always present | The shim and TOML parsing are poor fits for it. Hard to test. |
| C. Python package published to an index and installed with a tool such as pipx | Standard upgrade path | A build and publish step, and an installer the user must have. More than a prototype needs. |
| D. Go or Rust binary | Single static binary | A toolchain, and a release step for each platform. Departs from the stated constraint. |

## Decision

Option A. The file starts with a `python3` shebang line. A minimal packaging file can be added later to offer option C without changing the code (an open question).

Install and first-run steps, which are the ease-of-use criterion for this decision:

1. Get the file: clone the repository or download the single file.
2. Put it on `PATH` as `ccx`, for example `install -m 755 ccx <directory-on-PATH>`.
3. Run `ccx init`. It writes `~/.config/ccx/config.toml` (or under `XDG_CONFIG_HOME`) with placeholders, and refuses to overwrite an existing file.
4. Edit three values: `local.base_url`, `local.model`, `local.token_file`.
5. Run `ccx local --dry-run` to see the environment and command line, with the credential masked.
6. Run `ccx local`.

Tests load the file by path, so a single file does not block testing. The shim's rewrite and the config resolution are plain functions in the file.

## Consequences

- `ccx` works wherever Python 3.11 or later and `claude` are present. Older Pythons need a TOML reader, which is outside the standard library before 3.11. Whether to support them is an open question.
- No dependency to audit or update. No lockfile.
- One file means one place to read the whole tool, and one place to conflict in changes. If it passes a size where that hurts, it becomes a package with the same entry point.
- The name `ccx` can collide with another program of that name on the user's `PATH` (see [ADR 0006](0006-command-name-ccx.md)).

## Revisit when

A Python older than 3.11 matters; users want `pipx` or a distribution package; or the file becomes too large to read in one sitting.
