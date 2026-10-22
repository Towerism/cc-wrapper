# ccx architecture

`ccx` is a small command-line wrapper around Claude Code (`claude`). It runs `claude` against Anthropic's models, or against a locally hosted model that serves an Anthropic-compatible `/v1/messages` endpoint, while the user keeps one copy of their projects, history and config. This document is for the owner and for anyone changing `ccx` later. It says what `ccx` is meant to do, how the parts fit together, how it will be tested, and what is still undecided.

**Status:** design only, written 2026-10-22. Nothing here is implemented in this repository yet. `ccx` is an exploratory prototype: its command line, config keys and file layout can change between versions without notice, and the documents are written to make such changes cheap (see [ADR 0007](adr/0007-prototype-status-and-unstable-interface.md)). The decisions are in [docs/adr/](adr/) and the undecided items, each with what would settle it, are in [open-questions.md](open-questions.md).

## Summary

- `ccx` picks the backend by setting environment variables for one `claude` process. A running session cannot change backend. To switch, exit and resume the session in the other mode.
- Local mode shares the normal Claude config directory, so skills, memory, permissions and `/resume` history are the same in both modes. A sandbox option uses a separate directory instead, for experimenting with the config itself.
- `ccx` assumes nothing about the local server. The base URL, the credential file and the model name are supplied by the user. Before launching, it checks that the server answers.
- Some servers reject a `system` message that is not first in the `messages` array, and Claude Code sends one. `ccx` can start a small pass-through proxy, the shim, that rewrites such messages. The shim ships in this repository and runs only when needed.
- `ccx` is one Python 3 standard-library file. It needs `claude` and nothing from the local server's own tooling.

## Goals and non-goals

Goals:

1. Switch between Claude and a local model without duplicating projects, history or config.
2. Make the local-model setup safe by default: no telemetry, no nonessential traffic, no auto-updates, a permission mode that does not hand safety decisions to a weaker model, and a compaction window sized for a smaller context.
3. Keep every server detail in user configuration, and fail early with a clear message when the server is not there.
4. Be easy to install and configure: one file to install, one command to write a config template, one command to see what will happen.
5. Stay easy to change. Record what is open instead of locking an interface in early.

Non-goals:

- Installing, starting, stopping or tuning a model server. `ccx` does not know what engine or service manager is in use.
- Choosing a model, a quantization or a context size.
- Switching model inside a running session.
- Replacing `claude`'s own configuration. `ccx` sets environment variables and passes arguments through; it does not edit Claude Code's settings files.
- Supporting Claude Code alternatives. Only `claude` is wrapped.

## Modes

| Command | What it does | Config directory |
|---|---|---|
| `ccx [args]` | Runs `claude [args]` unchanged. `ccx` adds nothing. | The normal one |
| `ccx local [ccx flags] [claude args]` | Runs `claude` against the local model from the config file. | The normal one, so everything is shared |
| `ccx local --sandbox [claude args]` | The same, with an isolated config directory. | A separate directory (see [Sandbox mode](#sandbox-mode-and-its-known-leaks)) |
| `ccx init` | Writes a config template with placeholders if none exists. | Not applicable |
| `ccx local --dry-run` | Prints the resolved settings and the `claude` command line, with the credential masked, and launches nothing. | Not applicable |
| `ccx shim` | Runs the shim. `ccx local` starts it; users do not normally run it. | Not applicable |

Every mode runs `claude` in the current working directory. Arguments that `ccx` does not own pass through to `claude` unchanged. The proposed rule for telling the two apart is in [ADR 0007](adr/0007-prototype-status-and-unstable-interface.md): `ccx` flags come directly after `local`, only a small closed list is recognised, the first token not on the list ends `ccx` flag parsing, and `--` ends it explicitly. The rule is an open question until tried.

The candidate shape from the request is kept with two challenges, both recorded as open questions, not settled:

- The word `local` suggests the model runs on the same machine. The server could be elsewhere on a network. The config table is named `local` for now so a second backend could later be added as another table (see the open question on named backends).
- `--sandbox` exists only under `local`. A sandboxed plain mode would need its own Anthropic login in the separate directory, so it is not offered.

## Launch sequence for `ccx local`

1. Read the config file. A missing file or a missing required key stops the launch and names the key and the file.
2. Read the credential from the file named by `local.token_file`. A missing or unreadable file stops the launch and names the path. The credential is held in memory only (see [Secrets](#secrets)).
3. Check that the server answers (see [Reachability check](#reachability-check)).
4. Decide whether the shim is needed and, if so, start it (see [The shim](#the-shim)).
5. Build the environment and the `claude` argument list.
6. Run `claude`. Without the shim, `ccx` replaces itself with `claude` (`exec`), so nothing of `ccx` stays running. With the shim, `ccx` runs `claude` as a child, forwards signals to it, and stops the shim when `claude` exits.

## Configuration model

`ccx` reads one TOML (a plain-text configuration format) file, by default `~/.config/ccx/config.toml` (or under `XDG_CONFIG_HOME` when set). `ccx init` writes a template whose values are placeholders:

```toml
[local]
base_url   = "<base-url>"
model      = "<model-name>"
token_file = "<path-to-token-file>"
```

### What the user supplies

| Key | Required | Meaning | When absent |
|---|---|---|---|
| `local.base_url` | yes | Base URL of the server, as `claude` should see it | Launch stops |
| `local.model` | yes | The model name the server expects | Launch stops |
| `local.token_file` | yes | File holding the credential for the server | Launch stops |
| `local.compact_window` | no | Value for `CLAUDE_CODE_AUTO_COMPACT_WINDOW`, in tokens | The variable is not set, and `ccx` prints a one-line warning (see [ADR 0005](adr/0005-auto-mode-and-compaction-defaults.md)) |
| `local.max_output_tokens` | no | Value for `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | The variable is not set, and `ccx` prints a one-line warning |
| `local.permission_mode` | no | Permission mode to pass to `claude` | `default`, so auto mode is not used |
| `local.shim` | no | `auto`, `always` or `never` | `auto` |
| `local.sandbox_dir` | no | Where the sandbox config directory lives | A directory under the XDG data directory |

`ccx` has no default for any server detail: not the base URL, the model name, the credential location, a port, the inference engine or a service manager. The defaults above are all about `claude`'s behaviour or `ccx`'s own files.

### What `ccx` derives

For `ccx local`, `ccx` sets these variables in the environment of the `claude` process. It sets nothing in the user's shell.

| Variable | Value | Why |
|---|---|---|
| `ANTHROPIC_BASE_URL` | `local.base_url`, or the shim's address when the shim runs | Selects the backend |
| `ANTHROPIC_AUTH_TOKEN` | The contents of `local.token_file` | The credential. Observed: with this set, `claude` sends it as the `Authorization` header and sends no `x-api-key` header |
| `ANTHROPIC_API_KEY` | Unset | Stops a stored key from taking priority |
| `ANTHROPIC_MODEL` | `local.model` | The main model |
| `ANTHROPIC_DEFAULT_OPUS_MODEL`, `ANTHROPIC_DEFAULT_SONNET_MODEL`, `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `local.model` | Any alias `claude` resolves also reaches the local model |
| `CLAUDE_CODE_SUBAGENT_MODEL` | `local.model` | Subagents do not fall back to a Claude model |
| `DISABLE_TELEMETRY`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, `DISABLE_AUTOUPDATER` | `1` | No telemetry, no nonessential traffic, no auto-updates in local mode |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | `local.compact_window`, if set | A local context is smaller than Claude's |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | `local.max_output_tokens`, if set | Observed: without it `claude` asks for 32000 output tokens, which a small-context server may refuse; with it set to 4096, the request said 4096 |
| `CLAUDE_CONFIG_DIR`, `CLAUDE_HOME` | The sandbox directory, only with `--sandbox` | Isolation (see below) |

The variable names come from the earlier prototype and `claude`'s documentation. The three telemetry and update switches and `CLAUDE_CODE_AUTO_COMPACT_WINDOW` were not re-verified against Claude Code 2.1.299 in this design pass; the first implementation issue should verify them with the request-recording listener or the `claude` documentation.

## Reachability check

Before launching, `ccx` sends one cheap HTTP request to `local.base_url`. Any HTTP answer, including an error status such as 404, counts as reachable, because the aim is to tell "nothing is listening" from "something answered". A refused connection, a timeout, a name that does not resolve and a TLS failure each stop the launch with a distinct reason. The message names the config key and the config file, prints the base URL (it is not a secret), and says what to do:

```text
ccx: cannot reach the server at <base-url> (from local.base_url in <config-path>): connection refused.
     Start the server, or correct local.base_url.
```

The message never includes the credential. A `--no-check` flag to skip the check is proposed and listed as an open question.

## Switching backends and `/resume`

The backend is chosen from environment variables at process start, so a running session stays on its backend. Switching means exiting `claude` and starting `ccx` in the other mode, then using `/resume` (or `--resume <session>`).

Observed on 2026-10-22 with Claude Code 2.1.299 (details in [the evidence record](evidence/resume-across-backends.md)):

- A session started against one fake backend was resumed against a second fake backend with a different model name, using the same config directory and project directory. Resume worked headless (`--resume`) and interactively (the `/resume` picker listed the session, and the request after selecting it carried the earlier turns).
- The session identifier stayed the same, and the resumed request used the new model name.
- After resuming under a new model, `claude` added a `system` message announcing the model to the request. That is one source of the problem the shim solves.

What this does not show: resuming between the real Anthropic service and a real local model. That needs real credentials and a real server, and is marked unverified in the open questions.

## Sandbox mode and its known leaks

`ccx local --sandbox` sets `CLAUDE_CONFIG_DIR` to a separate directory, so skills, memory, permissions and history in it are independent of the normal config. It also sets `CLAUDE_HOME` to the same directory. The `Towerism/claude-config` repository's hooks read `CLAUDE_HOME` and fall back to the real Claude directory when it is unset, so setting only `CLAUDE_CONFIG_DIR` would let the hooks reach the real one.

The directory starts empty. To populate it, use that repository's `cfg deploy --target <dir>` command, which copies its managed files. Two limits apply and are recorded here so nobody assumes the isolation is complete:

1. **Known leak.** The deployed global `CLAUDE.md` names the real `~/.claude/skill-overlays/...` path as a fallback location. A session in the sandbox can therefore still read files outside it.
2. **Missing skills.** `cfg` manages only the paths in its host profile. Skills added by hand or synced from claude.ai are not copied, so a skill that works in the normal config can be missing in the sandbox.

Sandbox mode is for trying changes to the config. It is not a security boundary.

## The shim

### The problem

Claude Code can send a message with `role: "system"` after the first position of the `messages` array. Some chat templates (Qwen's, in the earlier attempt) reject this with `System message must be at the beginning`, and the first request fails with a 500. The request-recording listener showed, in Claude Code 2.1.299 with an empty config directory and no hooks, a request whose roles were `user, system`, where the second message was an 8.5 KB environment block. After a cross-backend resume the roles were `user, system, assistant, user, system`. So the failure does not need a `SessionStart` hook, as the earlier attempt assumed. A hook's added context is one more source, not the only one. This is a single-version observation (see [ADR 0002](adr/0002-optional-shim-in-repo.md)).

### What it does

The shim is a pass-through HTTP proxy on a loopback address:

- For `POST` requests to the messages path, it parses the JSON body, turns every `system` message that is not first into a `user` message, and merges adjacent `user` messages into one. When it merges, text added by the conversion goes after any `tool_result` blocks, because a user turn must begin with its tool results.
- It leaves the top-level `system` field and every other field alone, and forwards every other path and method unchanged.
- It streams the response back as it arrives, without buffering, and passes the status and headers through. It stops reading from the server when `claude` disconnects.
- It forwards the headers `claude` sent, including the credential, so it needs no copy of the secret.

### What it costs

The model sees the hook or environment text as part of a user turn, not as a system message. A model that weights system text more heavily will weight this text less. Merging also changes the turn structure that the model sees. The rewrite is deterministic, so a server's prompt cache still matches across requests whose earlier messages are unchanged.

### Where it lives, how it starts and stops, and when it is needed

The owner decided on 2026-10-22 that the shim ships in this repository ("the shim does ship in this repo"). It is the `ccx shim` subcommand of the same single file, so installing `ccx` installs the shim and no second program or service is involved.

- **Start.** `ccx local` starts one shim per session as a child process, listening on a loopback address with an operating-system-chosen port. The shim writes its address to the pipe `ccx` reads it from, so no port is fixed anywhere.
- **Stop.** `ccx` stops the shim when `claude` exits. The shim also exits if its input pipe closes, so a crashed `ccx` leaves no orphan. There is no shared daemon, no service manager and no state file to go stale.
- **Decision.** `local.shim` is `auto`, `always` or `never`. In `auto`, before the first launch for a given base URL and model, `ccx` sends one small probe request containing a `system` message after a `user` message. If the server accepts it, the shim is not used. If the server rejects it with the known `System message must be at the beginning` text, the shim is used. Any other failure leaves the decision unknown: `ccx` launches without the shim and prints a warning that names `local.shim = "always"`. The result is cached per base URL and model in `ccx`'s state directory. The probe is the only place `ccx` relies on a server behaviour, and it is the one failure mode already known.

The probe design is proposed and not yet tried against a real server; it is listed in the open questions.

## Secrets

The credential is read from the file named by `local.token_file` at launch, passed to `claude` through its environment, and never written anywhere by `ccx`. `--dry-run` shows it masked. Error messages name the file and never print its contents. `ccx` does not store, cache or log the credential, and the shim cache and logs hold none.

## Test strategy

The pyramid has four layers. Most logic is pure and sits at the bottom.

| Layer | What it covers | Determinism |
|---|---|---|
| Unit | The shim's message rewrite as a pure function (leading system kept, later system turned to user, adjacent users merged, string and block content, tool results first, other fields untouched, repeated rewrite changes nothing, empty list). Config and argument resolution as a pure function from config and arguments to environment and command line, including `--` handling and the masked credential. | No I/O, no clock |
| Integration | The shim in front of a fake upstream on loopback with operating-system-chosen ports: the upstream sees the rewritten body, streamed chunks reach the client one by one, the status and headers pass through, a client disconnect closes the upstream request, and the shim exits when its input pipe closes. The reachability failures against a closed port the test itself opened and closed. `--dry-run` output compared with a stored expectation. | Ports from the operating system; chunk order enforced with events the fake upstream waits on, never with sleeps |
| Request recording | A real `claude -p` run against a throwaway listener that records the request body and headers: which roles appear and where, that the `Authorization` header is the configured credential, the model and output-token values, and that with the shim in front no non-leading `system` message reaches the upstream. The test records the `claude` version and skips when `claude` is missing. | The listener replies with a fixed message; the test environment is built from an allowlist, not inherited |
| Interactive | A Python `pty` session that starts `claude` through `ccx`, handles the folder-trust prompt, sends `/skills` and `/resume`, and checks the screen text. | The test waits for expected screen text with a time limit instead of fixed sleeps; the environment is an allowlist; the config directory is temporary |

The headless `claude -p` check and the interactive check are not interchangeable. In the earlier attempt a headless run listed a skill that the interactive session did not have. In this design pass the two also differed in ways that would hide a bug: headless runs skip the folder-trust dialog, and an inherited `CLAUDE_CODE_*` variable stopped interactive sessions from being saved at all while headless runs saved normally. Only the interactive run is what the user sees, so only it can settle a question about `/skills` or `/resume`.

The public-repo hygiene scan (below) is the fifth check and runs before every push.

Coverage is set by risk, not by line count. The shim's rewrite function is the highest-risk logic, so every branch of it has a test. The launch sequence is covered by the resolution unit tests and the recording tests. The interactive layer stays small: a handful of journeys, because it is the slowest and the most fragile.

## Public-repo hygiene

This repository is public. Nothing committed, and no commit message, pull request text or issue text, may contain a home-directory path, a host name, a port number, an API key or token, or an email address, with these exceptions: `~`-relative paths to documented Claude Code locations, links under this owner's GitHub account and Claude Code's documentation sites, and the GitHub noreply addresses used for commit identities and the commit co-author trailer.

- Commits use the owner's GitHub noreply identity.
- Every example value is a placeholder in angle brackets, never a real or plausible host, port or model name.
- Machine-specific values live in the user's config file, outside the repository.
- A manual scan runs before each push and before a pull request is opened, over the added lines, the commit messages and author fields, and the PR and issue text. The exact patterns are in the approved spec on [issue 1](https://github.com/Towerism/cc-wrapper/issues/1); whether they become a committed script is an open question.
- Nothing is pushed to `main`; changes arrive through pull requests.

## Related

- [ADRs](adr/) record each decision with its status and evidence.
- [open-questions.md](open-questions.md) lists what is undecided and what would settle each item.
- [evidence/resume-across-backends.md](evidence/resume-across-backends.md) records the experiments cited above.
