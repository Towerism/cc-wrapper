# ADR 0006: The command is named `ccx`

**Status:** proposed, 2026-10-22. It becomes accepted when the owner approves it.

The interface is unstable ([ADR 0007](0007-prototype-status-and-unstable-interface.md)); undecided items are in [open-questions.md](../open-questions.md).

The tool is installed as `ccx`, not `cc`. This record is for anyone who wonders why the command has that name or wants to change it.

## Context

`cc` is the standard name of the C compiler on Unix-like systems, so a tool called `cc` earlier on `PATH` would shadow it and break builds. The owner chose `ccx` for that reason.

While checking the name for this record, one more collision turned up: CalculiX, a finite-element solver, installs an executable called `ccx`. A user with CalculiX installed has to arrange `PATH` order or pick another name for this tool. This was found by recall, not by testing on a machine with CalculiX, so it is unverified.

## Evidence

None measured. The compiler collision is general knowledge about the name `cc`. The CalculiX collision is from recall and untested.

## Options

| Option | For | Against |
|---|---|---|
| A. `ccx` | Short. Chosen by the owner. No shadowing of the compiler. | Same name as the CalculiX solver |
| B. `cc` | Shortest | Shadows the C compiler |
| C. A longer name such as `cc-wrapper` | No known collision | Slower to type. The repository already has that name. |

## Decision

Option A, with the collision listed as an open question. The installation steps in [ADR 0004](0004-language-and-packaging.md) let the user choose the installed file name, so a different name changes only the destination file name in the install command and needs no code change. Documentation refers to the command as `ccx` throughout.

## Consequences

- Users with CalculiX installed install under another name or put the directories in a different `PATH` order.
- `ccx init` and error messages print the name `ccx`. If a user renames the file, those messages stay `ccx`. Printing the invoked name is an open question.

## Revisit when

A user reports the collision, or the owner wants a different name.
