# TensorCLI

[English](README.md) | [简体中文](README.zh-CN.md)

A command-line interface and agent-facing tool layer for the **Tensor** software family: **TensorReading** (literature management and reading) and **TensorWriting** (academic writing).

TensorCLI lets you, your scripts, and AI agents (Claude Code, Copilot, Codex, MCP clients, and others) search, read, import, and cite from the same local libraries the desktop apps use, without opening the GUI.

> **Status: planning.** This repository currently contains the project requirements only. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) (in Chinese). No functionality is implemented yet.

## Goals

- One command, `tensor`, with a subcommand per app: `tensor reading ...`, `tensor writing ...`.
- Stable, scriptable output (`--json`) so that shell scripts and agents can parse results reliably.
- First-class agent integration: an MCP server and a ready-to-use agent skill.
- Safe by default: reads never modify the library; writes go through each app's official local API.

## Planned usage

The commands below are a design sketch and may change.

```bash
# TensorReading
tensor reading status                          # is the app running, where is the library
tensor reading search "deep brain stimulation" --limit 10 --json
tensor reading get <item_key> --fulltext       # metadata, PDF path, AI note
tensor reading collections
tensor reading import --doi 10.48550/arXiv.1706.03762 --collection "To read"
tensor reading cite <item_key> --style apa

# TensorWriting
tensor writing status
tensor writing list
# further commands: to be defined once the TensorWriting interface is confirmed

# Agent integration
tensor mcp serve                               # MCP server over stdio
tensor skill install --target claude           # install the agent skill
```

## How it works

TensorCLI talks to the Tensor apps through their local interfaces on your machine. Nothing is sent to a remote service.

| Interface | Used for | Availability |
|---|---|---|
| Local HTTP API (TensorReading: `127.0.0.1:23120`) | Metadata search, citations, all writes (imports) | App must be running |
| Library storage folder (read-only SQLite and files) | Full-text PDFs, AI notes, richer queries | Always, even if the app is closed |

## Roadmap

| Milestone | Scope |
|---|---|
| M0 | Repository, README, requirements (this commit) |
| M1 | CLI skeleton, config, `tensor reading` read commands |
| M2 | `tensor reading import`, duplicate check, PDF attach |
| M3 | MCP server and agent skill |
| M4 | `tensor writing` commands |
| M5 | Packaging and releases (Windows first, then macOS and Linux) |

Details and acceptance criteria are in [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md).

## Contributing

Issues and discussions are welcome. Please read the requirements document before proposing new commands so that naming and output conventions stay consistent.

## License

[MIT](LICENSE)
