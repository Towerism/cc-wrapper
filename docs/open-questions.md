# Open questions

These are the points the design leaves undecided. Each says what would settle it. The owner decides; nothing here is settled until a decision is recorded in an [ADR](adr/). Written 2026-10-22.

| # | Question | What would settle it |
|---|---|---|
| 1 | **Does `/resume` work between the real Anthropic service and a real local model?** Checked only between two fake backends ([evidence](evidence/resume-across-backends.md)). | One run with real credentials: start a session in plain mode, exit, resume it with `ccx local` against a real server, and the reverse. Record the result in the evidence file. |
| 2 | **Does the `system`-message problem appear in other Claude Code versions, or only in the one tested?** It appeared with no hooks in 2.1.299. | The request-recording test run against two or more `claude` versions, with the version recorded in the result. |
| 3 | **Is the shim probe reliable?** The probe in [ADR 0002](adr/0002-optional-shim-in-repo.md) was designed, not run against a real server. It relies on the known error text. | Run it against at least one server that rejects the message and one that accepts it. If the error text differs, change the matching rule or fall back to the config switch. |
| 4 | **How does `ccx local` tell its own flags from `claude` arguments?** The proposed rule (flags directly after `local`, a closed list, `--` ends them) is in [ADR 0007](adr/0007-prototype-status-and-unstable-interface.md). | Using it for a while. A real `claude` argument that collides with a `ccx` flag, or a user who is surprised by the rule, reopens it. |
| 5 | **Is `ccx` also a subcommand of other tools?** | The owner's decision after the standalone tool has been used. The standalone `ccx` does not block it. |
| 6 | **Should the config support more than one backend?** The key table is named `local` so named tables could be added. | A real need for a second backend, such as two servers or a remote one. |
| 7 | **What about a server that needs no credential?** This version requires `local.token_file`. | A real server of that kind. The likely answer is an explicit setting that supplies a non-secret placeholder, shown in `--dry-run`. |
| 8 | **Do settings in the shared config directory conflict with the variables `ccx` sets?** A settings file can set a model or environment variables. | A test that puts a model and an environment entry in a temporary settings file and checks which value `claude` uses. |
| 9 | **Do the telemetry, nonessential-traffic, update and compaction-window variable names still match Claude Code?** They came from the earlier prototype. Only the output-token variable was confirmed. | The request-recording test, or Claude Code's documentation, checked at implementation time. |
| 10 | **Does auto mode work against a local endpoint when a user turns it on?** | A run with auto mode on against a server that handles the classifier requests, and one against a server that does not. |
| 11 | **Python versions below 3.11.** `tomllib` is not in their standard library. | A user on an older Python. Options: a small TOML subset reader in the file, or require 3.11. |
| 12 | **Install channels.** The first version installs by copying one file. | Demand for a package index or distribution package. Packaging metadata would be added without code changes. |
| 13 | **Does the name `ccx` collide with CalculiX on the user's machines?** ([ADR 0006](adr/0006-command-name-ccx.md)) | `command -v ccx` on machines that have the solver, or a report from a user. |
| 14 | **Should the hygiene scan become a committed script and a CI job?** It is a manual command in this version. | The first implementation issue. A script that exits non-zero on an unexplained match, run by CI, would replace the manual step. |
| 15 | **Skills missing from the sandbox.** Hand-added and claude.ai-synced skills are not copied by `cfg deploy --target`. | A change in `Towerism/claude-config`, or a documented manual step. It is outside this repository. |
| 16 | **Flag details.** Whether to offer `--no-check` and `--no-shim`, how to reset the cached shim probe result, and whether error messages print the name the tool was invoked under. | Use. Each is small and can be decided in the implementation issue. |
