# ADR 0007: `ccx` is a prototype with an unstable interface

**Status:** proposed, 2026-10-22. It becomes accepted when the owner approves it.

Nothing about the command line, the config keys or the file layout is promised. Breaking changes between versions are fine. Each document says what is still open, and [open-questions.md](../open-questions.md) lists it.

## Context

The owner described `ccx` as an exploratory prototype whose command-line surface is not settled, and asked for a design that is easy to change over one that locks an interface in early. Several choices are guesses until the tool is used: how to tell a `ccx` flag from a `claude` argument, whether `local` is the right word, whether one backend is enough.

## Options

| Option | For | Against |
|---|---|---|
| A. State that the interface is unstable; keep the surface small | Cheap to change. Honest. | Users cannot rely on scripts that call `ccx` |
| B. Version the interface from the start | Stable for scripts | Costs design effort before the uses are known |

## Decision

Option A, with these working rules for the first version:

- **Subcommands:** `ccx` (plain), `ccx local`, `ccx init`, `ccx shim`. Nothing else is added without a reason recorded here.
- **Flags of `ccx local`:** `--sandbox`, `--dry-run` and, as a proposal, `--no-check` and `--no-shim`. They come directly after `local`. The first token not on this closed list ends `ccx` flag parsing, and `--` ends it explicitly. Everything after goes to `claude`.
- **Config:** one TOML (plain-text configuration) file, keys under a `local` table. Unknown keys are an error, so a misspelling is caught.
- **Changes:** a release may rename any of these. The changelog says what changed; no deprecation period is promised.

## Consequences

- The architecture document and the ADRs describe the interface as a proposal.
- Scripts that call `ccx` pin a version.
- If the flag rule causes a collision with a real `claude` argument, the rule changes, not `claude`.

## Revisit when

The interface has been used for a while and the open questions about flags and named backends are answered; then a stable interface can be written down.
