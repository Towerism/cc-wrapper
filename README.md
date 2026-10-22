# cc-wrapper

`ccx` is a small command-line wrapper around Claude Code (`claude`) that lets one person switch between Claude and a locally hosted open-weights model behind an Anthropic-compatible `/v1/messages` endpoint, without keeping two copies of their projects, history or config. This repository is at the design stage: nothing is implemented yet, and the interface will change. Start with the [architecture document](docs/architecture.md), read the decisions in the [ADRs](docs/adr/), and see what is still undecided in the [open questions](docs/open-questions.md).
