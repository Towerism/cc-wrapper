# Evidence: `/resume` across two backends, and what a session sends

This record is for anyone who needs to know how the claims in [the architecture document](../architecture.md) and the [ADRs](../adr/) were checked, or who wants to repeat the checks. It lists commands as placeholders. No transcript is committed, because the real ones contain local paths, addresses and session identifiers.

**Date:** 2026-10-22. **Claude Code version:** 2.1.299. **Platform:** Linux, Python 3 standard library.

## What the experiments prove, and what they do not

They prove that one config directory and one project directory can be resumed under two different backend environments, and they show exactly what `claude` puts in each request. They do not prove anything about the real Anthropic service, a real local model, or any server's chat template. Step 4 of the method used the real config directory; the spec planned a temporary one, and the deviation is recorded in the pull request. The listeners return a fixed reply. A temporary config directory has no Anthropic login, so plain mode with real credentials was not run (see the open questions).

## Method

1. A throwaway listener: a few lines of Python standard library, bound to a loopback address and an operating-system-chosen port, that appends each request's path, its `authorization` and `x-api-key` headers, and its JSON body to a file, then answers `/v1/messages` with a fixed streamed reply (or a fixed JSON reply when `stream` is false). Two instances played backend A and backend B and each answered with its own name.
2. A temporary config directory and a temporary project directory.
3. An environment built for each run: `CLAUDE_CONFIG_DIR=<temp-config>`, `ANTHROPIC_BASE_URL=<listener-address>`, `ANTHROPIC_AUTH_TOKEN=<dummy>`, `ANTHROPIC_MODEL=<model-a or model-b>`, with `ANTHROPIC_API_KEY` unset and telemetry, nonessential traffic and updates switched off.
4. One check used the owner's real config directory, which holds a Claude login: a headless `claude -p` run with `--no-session-persistence`, `ANTHROPIC_AUTH_TOKEN` set to a dummy value and the base URL pointing at a listener, so no real service was contacted. That run was not isolated from the real config directory, and `claude` may still have updated its own state files there; only the request headers were inspected. Nothing in this check supports a claim beyond that header.
5. For the interactive checks, a Python `pty` session that starts `claude`, answers the folder-trust prompt by moving the selection to "Yes" and pressing Enter, sends text and slash commands, and reads the screen.

## Results

### Headless resume

1. `claude -p "remember the word <word>"` against backend A printed a session identifier.
2. `claude -p "what word did I ask you to remember?" --resume <session-id>` against backend B returned backend B's reply, with the same session identifier.
3. Backend B's recorded request carried the model name for B, the first user turn containing the word, backend A's earlier reply as an `assistant` turn, and the new question.

### Interactive resume

1. An interactive session against backend A received the same instruction and was exited with `/exit`.
2. A new interactive session against backend B ran `/resume`. The picker listed the session from backend A.
3. Selecting it and asking the question produced backend B's reply, and backend B's recorded request carried the earlier turns.

### What a session sends

- **Roles.** A fresh headless session with an empty config directory and no hooks sent the roles `user, system`. The `system` message was the environment block, about 8.5 KB. The resumed session sent `user, system, assistant, user, system`; the last one announced the model name. The top-level `system` field was also present, in three blocks.
- **Credential.** In the run described in method step 4, the `Authorization` header equalled the configured token and no `x-api-key` header was sent. So in that run the stored login did not reach the listener. This is one version and one fake backend.
- **Size.** The first request body of a headless run with a two-letter prompt was about 87,000 characters, of which about 58,000 were the definitions of 21 tools and about 6,000 the top-level system text. At roughly four characters per token that is on the order of 20,000 tokens before the user's first word. This is an estimate; the point is that a model context in the low tens of thousands of tokens leaves little room.
- **Output tokens.** The request asked for 32000 output tokens by default. With `CLAUDE_CODE_MAX_OUTPUT_TOKENS` set to 4096 it asked for 4096.
- **Auto mode.** The interactive status line showed auto mode on at launch against the listener. With `--permission-mode default` it did not. A notice also said the session was not eligible for a change to how auto mode's classifier requests are billed, because the requests go through a gateway that does not support it. That implies classifier requests use the same gateway as the session, which was not tested. This status line appeared in an environment with other inherited `CLAUDE_CODE_*` variables, so the source of the default is not isolated. See [ADR 0005](../adr/0005-auto-mode-and-compaction-defaults.md).
- **Harness effects.** The interactive session did not save its transcript until `CLAUDE_CODE_FORCE_SESSION_PERSISTENCE=1` was set, because the surrounding environment already held other `CLAUDE_CODE_*` variables. A test that does not control its environment can fail for reasons unrelated to the code under test.

## Recorded requests, sanitized

Roles and selected fields from the recorded requests. Addresses, paths and session identifiers are replaced.

| Step | Backend | Model field | Message roles | Header check |
|---|---|---|---|---|
| Headless, first turn | A | `<model-a>` | `user, system` | not inspected |
| Headless, resumed with `--resume <session-id>` | B | `<model-b>` | `user, system, assistant, user, system` | not inspected |
| Interactive, resumed through `/resume`, then one question | B | `<model-b>` | `user, system, assistant, user, system` | not inspected |
| Headless, real config directory, dummy token | C | `<model-c>` | `user, system` | `Authorization` equals `Bearer <dummy>`; no `x-api-key` |

In the resumed requests the first `user` turn and the earlier `assistant` turn (`reply from A`) carried the original instruction and backend A's reply. The listener's own code is not committed: it binds to a loopback address, so it would put an address in a public repository, and the checks it supports are replaced by the tests in [the test strategy](../architecture.md#test-strategy) once `ccx` exists.

## Not verified

- Resume between the real Anthropic service and a real local model.
- Whether the mid-array `system` message appears in other Claude Code versions or only when the surrounding environment adds it. (The owner separately reported, without a test here, that two Qwen models reject it with no hooks configured.)
- The names of the telemetry, nonessential-traffic, update and compaction-window variables on this version.
- That a real server accepts the history Claude Code produced, including thinking blocks with signatures, tool-use identifiers and tool names from Model Context Protocol (MCP) servers.
- Whether auto mode is the default in a clean environment.

## To repeat

Write the listener and the `pty` driver as described, run them with temporary directories, and compare against the results above. When `ccx` has tests, the request-recording and interactive layers of [the test strategy](../architecture.md#test-strategy) replace this manual record.
