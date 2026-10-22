# ADR 0003: Server details are configuration, never defaults

**Status:** proposed, 2026-10-22. It becomes accepted when the owner approves it.

The interface is unstable ([ADR 0007](0007-prototype-status-and-unstable-interface.md)); undecided items are in [open-questions.md](../open-questions.md).

The base URL, the credential file and the model name are values the user supplies. `ccx` has no default for them, or for a port, a model alias, a key location, an inference engine or a service manager. It checks that the server answers before it launches `claude`.

## Context

The earlier attempt hardcoded a default listening address for its shim and was tied to one server setup. This repository is public and is meant to work with any server that serves an Anthropic-compatible `/v1/messages` endpoint. The one thing `ccx` may know about a server is the failure described in [ADR 0002](0002-optional-shim-in-repo.md).

## Evidence

None measured. This is a design choice that follows from the owner's rule that `ccx` assumes nothing about the server. The earlier prototype's hardcoded default address is the counter-example, from reading its source.

## Options

| Option | For | Against |
|---|---|---|
| A. No defaults for server details; fail with a clear message when one is missing | Works with any server. Nothing machine-specific in the repo. | The user must write three values before first use |
| B. Defaults for a common setup | Fewer first-run steps | Wrong for everyone else, and a default address or model name in a public repo is a machine-specific value |
| C. Auto-discover the server | No setup | Needs knowledge of engines and service managers, which `ccx` must not have |

## Decision

Option A. To keep first use short, `ccx init` writes a config template whose values are placeholders, and `ccx local --dry-run` shows what will run before anything starts. The config file has three required keys: `local.base_url`, `local.model` and `local.token_file`.

The reachability check treats any HTTP answer as reachable. A refused connection, a timeout, a name that does not resolve and a TLS failure stop the launch with distinct messages that name the config key and the config file. The credential is read from the named file at launch and never stored, cached, logged or echoed.

## Consequences

- First use takes three edits, not zero. This is the price of not assuming.
- Documentation and tests use placeholders everywhere, and the hygiene scan in [the architecture document](../architecture.md#public-repo-hygiene) checks it.
- A server that needs no credential still needs a credential file in this version. The handling of such servers is an open question.
- Tests that need a server use a fake one on a loopback address with an operating-system-chosen port.

## Revisit when

Users ask for more than one backend (named tables in the config file), or for a server that needs no credential.
