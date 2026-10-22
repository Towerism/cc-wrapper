# ccx architecture

`ccx` is a small command-line wrapper around Claude Code (`claude`). It runs `claude` against Anthropic's models, or against a locally hosted model that serves an Anthropic-compatible `/v1/messages` endpoint, while the user keeps one copy of their projects, history and config. This document is for the owner and for anyone changing `ccx` later. It says what `ccx` is meant to do, how the parts fit together, how it will be tested, and what is still undecided.

**Status:** design only, written 2026-10-22. Nothing here is implemented in this repository yet. `ccx` is an exploratory prototype: its command line, config keys and file layout can change between versions without notice, and the documents are written to make such changes cheap (see [ADR 0007](adr/0007-prototype-status-and-unstable-interface.md)). The decisions are in [docs/adr/](adr/) and the undecided items, each with what would settle it, are in [open-questions.md](open-questions.md).

## Summary

- `ccx` picks the backend by setting environment variables for one `claude` process. A running session cannot change backend. To switch, exit and resume the session in the other mode. That workflow was checked only between two fake backends (see [Switching backends](#switching-backends-and-resuming)).
- Local mode shares the normal Claude config directory, so skills, memory, permissions and `/resume` history are the same in both modes. A sandbox option uses a separate directory instead, for experimenting with the config itself.
- `ccx` assumes nothing about the local server. The base URL, the credential file and the model name are supplied by the user. Before launching, it checks that the server answers.
- Some servers reject a `system` message that is not first in the `messages` array, and Claude Code 2.1.299 sent one even with no hooks configured. `ccx` can start a small pass-through proxy, the shim, that rewrites such messages. The shim ships in this repository and runs only when needed.
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
| `ccx [args]` (plain, or passthrough) | Runs `claude [args]` unchanged. `ccx` adds nothing. | The normal one |
| `ccx local [ccx flags] [claude args]` | Runs `claude` against the local model from the config file. | The normal one, so everything is shared |
| `ccx local --sandbox [claude args]` | The same, with an isolated config directory. | A separate directory (see [Sandbox mode](#sandbox-mode-and-its-known-leaks)) |
| `ccx init` | Writes a config template with placeholders if none exists. | Not applicable |
| `ccx local --dry-run` | Prints the resolved settings and the `claude` command line, with the credential masked, and launches nothing. | Not applicable |
| `ccx shim` | Runs the shim. `ccx local` starts it; users do not normally run it. | Not applicable |

Every mode runs `claude` in the current working directory. Arguments that `ccx` does not own pass through to `claude` unchanged, with one exception: in local mode `ccx` may add `--permission-mode` (see [ADR 0005](adr/0005-auto-mode-and-compaction-defaults.md)). The subcommand names `local`, `init` and `shim` are taken by `ccx`, so a `claude` prompt or subcommand with one of those names must be run as plain `ccx -- <name>` or directly as `claude`; whether `--` is consumed or forwarded is an open question. The proposed rule for telling the two apart is in [ADR 0007](adr/0007-prototype-status-and-unstable-interface.md): `ccx` flags come directly after `local`, only a small closed list is recognised, the first token not on the list ends `ccx` flag parsing, and `--` ends it explicitly. The rule is an open question until tried.

The candidate shape from the request is kept with two challenges, both recorded as open questions, not settled:

- The word `local` suggests the model runs on the same machine. The server could be elsewhere on a network. The config table is named `local` for now so a second backend could later be added as another table (see the open question on named backends).
- `--sandbox` exists only under `local`. A sandboxed plain mode would need its own Anthropic login in the separate directory, so it is not offered.

## Launch sequence for `ccx local`

1. Read the config file. A missing file or a missing required key stops the launch and names the key and the file.
2. Read the credential from the file named by `local.token_file`. A missing or unreadable file stops the launch and names the path. The credential is held in memory only (see [Secrets](#secrets)).
3. Check that the server answers (see [Reachability check](#reachability-check)).
4. Decide whether the shim is needed and, if so, start it (see [The shim](#the-shim)).
5. Build the environment and the `claude` argument list (see [Inherited environment](#inherited-environment)).
6. Run `claude`. Without the shim, `ccx` replaces itself with `claude` (`exec`), so nothing of `ccx` stays running. With the shim, `ccx` runs `claude` as a child and stops the shim when `claude` exits (see [Process model](#process-model-with-the-shim)).

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
| `local.sandbox_dir` | no | Where the sandbox config directory lives | A directory under the XDG data directory (`~/.local/share/ccx/` unless `XDG_DATA_HOME` is set) |

`ccx` keeps its own state, the shim probe cache, in `~/.local/state/ccx/` (or under `XDG_STATE_HOME`). It holds no credential.

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

## Inherited environment

`ccx local` starts from the environment it was given and then makes these changes, so a stray variable cannot send a local session to the wrong place:

- **Set** the variables in the table above.
- **Remove** a proposed scrub list that points `claude` at another backend or adds headers: `ANTHROPIC_API_KEY`, `ANTHROPIC_CUSTOM_HEADERS`, `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX` and older model aliases such as `ANTHROPIC_SMALL_FAST_MODEL`. The list is a proposal from memory, not checked against this version (open question 18).
- **Leave** `CLAUDE_CONFIG_DIR` alone in shared mode, because it may be how the user's normal config is found. In sandbox mode it is replaced.
- **Add** the shim's loopback host to `NO_PROXY` and `no_proxy` when the shim runs, so a proxy variable does not capture the traffic between `claude` and the shim.

The evidence shows the hazard is real: other `CLAUDE_CODE_*` variables in the surrounding environment stopped interactive sessions from being saved. Tests build the environment for `claude` from an allowlist, and a unit test covers the scrub list.

## Reachability check

Before launching, `ccx` sends one cheap `GET` to `local.base_url`, keeping any path prefix and following no redirects. Any HTTP answer, including an error status such as 404, counts as reachable, because the aim is to tell "nothing is listening" from "something answered". A refused connection, a timeout, a name that does not resolve and a TLS failure each stop the launch with a distinct reason. The message names the config key and the config file, prints the base URL (it is not a secret), and says what to do:

```text
ccx: cannot reach the server at <base-url> (from local.base_url in <config-path>): connection refused.
     Start the server, or correct local.base_url.
```

The message never includes the credential. A proxy or a captive portal that answers for every address would make an unreachable server look reachable; the check cannot tell. A self-signed certificate on a local network fails the TLS step; a `--no-check` flag is proposed for that and listed as an open question.

## Switching backends and resuming

The backend is chosen from environment variables at process start, so a running session stays on its backend. Switching means exiting `claude` and starting `ccx` in the other mode, then using `/resume` (or `--resume <session>`).

Observed on 2026-10-22 with Claude Code 2.1.299 (details in [the evidence record](evidence/resume-across-backends.md)):

- A session started against one fake backend was resumed against a second fake backend with a different model name, using the same config directory and project directory. Resume worked headless (`--resume`) and interactively (the `/resume` picker listed the session, and the request after selecting it carried the earlier turns).
- The session identifier stayed the same, and the resumed request used the new model name.
- After resuming under a new model, `claude` added a `system` message announcing the model to the request. That is one source of the problem the shim solves.

What this does not show: resuming between the real Anthropic service and a real local model. That needs real credentials and a real server, and is marked unverified in the open questions.

## Sandbox mode and its known leaks

`ccx local --sandbox` sets `CLAUDE_CONFIG_DIR` to a separate directory, so skills, memory, permissions and history in it are independent of the normal config. It also sets `CLAUDE_HOME` to the same directory. The `Towerism/claude-config` repository's hooks read `CLAUDE_HOME` and fall back to the real Claude directory when it is unset, so setting only `CLAUDE_CONFIG_DIR` would let the hooks reach the real one.

The directory starts empty. To populate it, use that repository's `cfg deploy --target <dir>` command, which copies its managed files. Three limits apply and are recorded here so nobody assumes the isolation is complete:

1. **Known leak.** The deployed global `CLAUDE.md` names the real `~/.claude/skill-overlays/...` path as a fallback location. A session in the sandbox can therefore still read files outside it.
2. **Missing skills.** `cfg` manages only the paths in its host profile. Skills added by hand or synced from claude.ai are not copied, so a skill that works in the normal config can be missing in the sandbox.

3. **Project files.** Every mode runs in the current working directory, so a `.claude/` directory, a `CLAUDE.md` or an `.mcp.json` in the project still applies in the sandbox.

Sandbox mode is for trying changes to the config. It is not a security boundary.

## The shim

### The problem

Claude Code can send a message with `role: "system"` after the first position of the `messages` array. Some chat templates (Qwen's, in the earlier attempt) reject this with `System message must be at the beginning`, and the first request fails with a 500. The request-recording listener showed, in Claude Code 2.1.299 with an empty config directory and no hooks, a request whose roles were `user, system`, where the second message was an 8.5 KB environment block. After a cross-backend resume the roles were `user, system, assistant, user, system`. So the failure does not need a `SessionStart` hook, as the earlier attempt assumed. A hook's added context is one more source, not the only one. The request shape is a single-version observation. The owner confirmed on 2026-10-22 that the rejection happens without hooks with two widely used Qwen models, `qwen3.8-27b` and `qwen3.6-35b` (see [ADR 0002](adr/0002-optional-shim-in-repo.md)); that confirmation is a report, not a test in this repository.

### What it does

The shim is a pass-through HTTP proxy on a loopback address:

- For `POST` requests to the messages path, it parses the JSON body, turns every `system` message that is not first into a `user` message, and merges adjacent `user` messages into one. When it merges, text added by the conversion goes after any `tool_result` blocks, because a user turn must begin with its tool results.
- It leaves the top-level `system` field and every other field alone, and forwards every other path and method unchanged.
- It streams the response back as it arrives, without buffering, and passes the status and headers through. It stops reading from the server when `claude` disconnects.
- It forwards the headers `claude` sent, including the credential, so it needs no copy of the secret.

### What it costs

The model sees the hook or environment text as part of a user turn, not as a system message. A model that weights system text more heavily will weight this text less. Merging also changes the turn structure that the model sees. The rewrite is deterministic, so a server's prompt cache still matches across requests whose earlier messages are unchanged.

### Where it lives

The owner decided on 2026-10-22 that the shim ships in this repository ("the shim does ship in this repo"). It is the `ccx shim` subcommand of the single `ccx` file, which sits at the repository root, so installing `ccx` installs the shim and no second program or service is involved.

### Process model with the shim

When the shim is not used, `ccx` replaces itself with `claude` and nothing of `ccx` stays running. When it is used:

- **Start.** `ccx local` starts one shim per session as a child, in its own session so that a terminal Ctrl-C does not reach it. The shim listens on a loopback address only, with an operating-system-chosen port, and writes that address as its first line on standard output. No port is fixed anywhere.
- **Lifetime.** `ccx` holds the write end of the shim's standard input. The shim reads it and exits on end of file, so if `ccx` dies for any reason the shim exits too. `ccx` also stops the shim when `claude` exits.
- **Signals.** `ccx` runs `claude` as a child in the foreground process group. While it runs, `ccx` ignores the interrupt signal the terminal sends on Ctrl-C (SIGINT), because `claude` receives it from the terminal itself and a forwarded second one would make the interface quit. It forwards the terminate and hang-up signals (SIGTERM and SIGHUP) to `claude`. It exits with `claude`'s exit status, or 128 plus the signal number when `claude` dies from a signal. Job control (Ctrl-Z) is not designed yet and is an open question.
- **No shared daemon.** There is no service manager, no discovery file and no stale process to find.

The integration tests cover each of these: the shim exits when `ccx` is killed, one Ctrl-C reaches `claude` once, and the exit status is propagated.

### When the shim is needed

`local.shim` is `auto`, `always` or `never`, with `auto` as the default. In `auto`, before the first launch for a given base URL and model, `ccx` sends two small requests, each limited to a few output tokens, using the credential:

1. A control request: a single `user` message.
2. A test request: a `user` message followed by a `system` message, the shape Claude Code 2.1.299 sends first.

The result:

| Control | Test | Meaning | Action |
|---|---|---|---|
| accepted | accepted | The server tolerates the shape (or silently drops the `system` text) | No shim |
| accepted | rejected with any 4xx or 5xx status | The shape is the problem | Use the shim |
| rejected, or no answer | any | The probe cannot tell, for instance a model still loading | Launch without the shim and warn, naming `local.shim = "always"` |

The known error text (`System message must be at the beginning`) is used only to make the diagnostic message more helpful. It is not the test, because other templates fail with other words.

What the probe costs and cannot see:

- Both requests are real inference requests. They cost tokens on a metered server and can force a model to load, so the probe uses a longer time limit than the reachability check.
- A server that accepts the shape but silently drops the `system` text passes the probe, and the hook and environment text is then lost without any error. The probe cannot detect that.
- It tests one position. Claude Code also sends `system` after later turns.
- The result is cached per base URL and model in `ccx`'s state directory, with a time limit after which it is probed again (proposed: seven days). A stale cached "no shim" is the dangerous case, because the user then sees a bare 500 inside `claude`. `local.shim = "always"` is the documented way out, and a flag to clear the cache is proposed.

The probe has not been tried against a real server (open question 3).

### Building the shim

Rules for whoever implements it:

- Bind to the loopback address only. Keep any path prefix that `local.base_url` has. Speak HTTP/1.1 and use chunked transfer or connection close, so streaming works under the standard library's HTTP server.
- Never log request or response bodies, and never log headers.
- Merging can move or drop `cache_control` markers. The rewrite keeps them on their original blocks.
- Anthropic's message format wants a user turn's `tool_result` blocks first. When a converted `system` message sits between an assistant `tool_use` turn and the user turn with its results, the merge puts the converted text after the results. That rule comes from the API documentation and was not tested here.
- A `system` message that is last in the list (the observed shape) is handled like any other non-leading one.

## Secrets

The credential is read from the file named by `local.token_file` at launch, passed to `claude` through its environment, and never written anywhere by `ccx`. `--dry-run` shows it as the fixed text `<redacted>`, not a partial mask. The file's trailing newline is stripped, and `ccx` warns when the file can be read by its group or by others. Error messages name the file and never print its contents. `ccx` does not store, cache or log the credential, and the probe cache and the shim's logs hold none.

`--dry-run` contacts nothing. It does not run the reachability check or the probe, so it reports the shim as `unknown (not probed)` unless a cached result exists. It prints the full `claude` argument list, which can include prompt text.

Known exposure: the credential sits in `claude`'s environment, so every shell command and hook that `claude` runs inherits it, and a command that prints the environment would put it in the session transcript. Because the config directory is shared, such a transcript could later be resumed in a plain session. A way to avoid this, giving `claude` a dummy token and letting the shim add the real one, is possible when the shim runs and is recorded as a trade-off in the open questions.

## Test strategy

Most logic is pure and sits at the bottom of the pyramid. There are six checks.

| Layer | What it covers | Determinism |
|---|---|---|
| Unit | The shim's message rewrite as a pure function (leading system kept, later system turned to user, adjacent users merged, string and block content, tool results first, other fields untouched, repeated rewrite changes nothing, empty list, a trailing `system` message, a `system` message between `tool_use` and `tool_result` turns). Config and argument resolution as a pure function from config and arguments to environment and command line, including `--` handling, the rule for adding `--permission-mode`, the scrub list and `NO_PROXY` handling, and the redacted credential. | No I/O, no clock |
| Integration | The shim in front of a fake upstream on loopback with operating-system-chosen ports: the upstream sees the rewritten body, streamed chunks reach the client one by one, the status and headers pass through, a client disconnect closes the upstream request, and the shim exits when its input pipe closes. The reachability failures against a closed port the test itself opened and closed. `--dry-run` output compared with a stored expectation. | Ports from the operating system; chunk order enforced with events the fake upstream waits on, never with sleeps. Cases also cover the signal and exit-status behaviour in [The shim](#the-shim) |
| Request recording | A real `claude -p` run against a throwaway listener that records the request body and headers: which roles appear and where, that the `Authorization` header is the configured credential, the model and output-token values, that `--permission-mode` is added only where the rule allows, and that with the shim in front no non-leading `system` message reaches the upstream. The test records the `claude` version. | The listener replies with a fixed message; the test environment is built from an allowlist, not inherited |
| Headless skills listing | A `claude -p` run through `ccx` that asks which skills are available. It confirms that the config directory (shared or sandbox) is the one `ccx` meant. It is a quick check, and it cannot show what the user sees. | Temporary config directory with known skills; fixed reply from the listener |
| Interactive | A Python `pty` session that starts `claude` through `ccx`, handles the folder-trust prompt, sends `/skills` and `/resume`, and checks the screen text. | The test waits for expected screen text with a time limit instead of fixed sleeps; the environment is an allowlist; the config directory is temporary |
| Hygiene scan | The text scan in [Public-repo hygiene](#public-repo-hygiene) over added lines, commit messages and PR text. | A pure text scan; no network, no clock |

The headless `claude -p` check and the interactive check are not interchangeable. In the earlier attempt a headless run listed a skill that the interactive session did not have. In this design pass the two also differed in ways that would hide a bug: headless runs skip the folder-trust dialog, and an inherited `CLAUDE_CODE_*` variable stopped interactive sessions from being saved at all while headless runs saved normally. Only the interactive run is what the user sees, so only it can settle a question about `/skills` or `/resume`.


The request-recording, headless and interactive checks need `claude`. When it is missing they skip on a developer machine and fail when the `CI` environment variable is set, so a continuous-integration run cannot pass without them. The hygiene scan runs before every push.

Coverage is set by risk, not by line count. The shim's rewrite function is the highest-risk logic, so every branch of it has a test. The launch sequence is covered by the resolution unit tests and the recording tests. The interactive layer stays small: a handful of journeys, because it is the slowest and the most fragile.

## Public-repo hygiene

This repository is public. Nothing committed, and no commit message, pull request text or issue text, may contain a home-directory path, a host name, a port number, an API key or token, or an email address, with these exceptions: `~`-relative paths to Claude Code's documented locations and this tool's own config and state directories, links under this owner's GitHub account and the `claude.com` and Claude Code documentation sites, and the GitHub noreply addresses used for commit identities and the commit co-author trailer.

- Commits use the owner's GitHub noreply identity.
- Every example value is a placeholder in angle brackets, never a real or plausible host, port or model name.
- Machine-specific values live in the user's config file, outside the repository.
- A manual scan runs before each push and before a pull request is opened, over the added lines, the commit messages and author fields, and the PR and issue text. The exact patterns are in the approved spec on [issue 1](https://github.com/Towerism/cc-wrapper/issues/1); whether they become a committed script is an open question.
- Nothing is pushed to `main`; changes arrive through pull requests.

## Related

- [ADRs](adr/) record each decision with its status and evidence.
- [open-questions.md](open-questions.md) lists what is undecided and what would settle each item.
- [evidence/resume-across-backends.md](evidence/resume-across-backends.md) records the experiments cited above.
