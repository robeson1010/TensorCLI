# TensorCLI

[English](README.md) | [简体中文](README.zh-CN.md)

`tensorx` is the command line for **TensorReading** (literature management and reading) and **TensorWriting** (LaTeX writing). You, your scripts and AI agents (Claude Code, Codex, Copilot and others) can use it to import papers, generate outlines and the knowledge graph, and compile LaTeX projects to PDF.

This repository holds only the download links, the usage guide and an agent skill. `tensorx` is installed together with TensorReading; there is nothing else to install.

## Download

| App | Download | Notes |
|---|---|---|
| TensorReading | <https://www.tensorx.xin> | Includes the `tensorx` command |
| TensorWriting | <https://www.tensorx.xin/tensorwriting> | Needed for `tensorx writing` |

After installing TensorReading and starting it once:

- **Windows**: the install folder is added to your user PATH. Open a new terminal and run `tensorx`.
- **macOS / Linux**: `~/.local/bin/tensorx` is created. If the shell cannot find it, add `~/.local/bin` to your PATH.

Check the installation:

```bash
tensorx --version
tensorx status
```

## Before you start

- The desktop apps do the work. If an app is not running, `tensorx` starts it (pass `--no-launch` to prevent that).
- **The app must be signed in to a TensorX account**, otherwise the command is refused (exit code 7). Sign in inside the app, or run:

```bash
tensorx reading login
tensorx writing login
```

- TensorWriting needs TeX Live installed (from its account menu).

## TensorWriting: compile a LaTeX project

```bash
tensorx writing compile ./my-paper                 # compile a project folder
tensorx writing compile ./my-paper/main.tex        # name the main file
tensorx writing compile ./my-paper -o out/paper.pdf
tensorx writing compile ./thesis --main chapters/main.tex --engine xelatex
tensorx writing open ./my-paper                    # open in the app without compiling
```

`compile` switches the TensorWriting window to the project, picks the main file and engine by the app's own rules, builds it, and writes the PDF:

- by default next to the main file with the same name (`main.tex` → `main.pdf`);
- or wherever `-o` says. The path must end in `.pdf` and its folder must exist.

| Option | Meaning |
|---|---|
| `--main <relative path>` | Main file; same as "Set as main file" in the app |
| `--engine pdflatex\|xelatex\|lualatex` | Engine for this build only; the project setting is unchanged |
| `--mode full\|quick` | Full (default) or quick compilation |
| `-o, --output <file.pdf>` | Where to write the PDF |
| `--log` | Also print the full compile log |

A failed build lists its errors as `file:line: error: message` and exits with code 6. Missing packages are downloaded automatically; progress goes to stderr.

## TensorReading: papers, outlines and the knowledge graph

```bash
# Import (duplicate check, metadata lookup, open-access PDF download)
tensorx reading import --doi 10.1038/s41586-021-03819-2
tensorx reading import --arxiv 1706.03762 --collection "Transformers" --tag to-read
tensorx reading import --pdf ./paper.pdf --outline      # local PDF, outline right away

# Outline a paper and add it to the knowledge graph
tensorx reading outline <item_key>
tensorx reading outline <key1> <key2> --force           # regenerate existing outlines

# Knowledge base (knowledge graph)
tensorx reading kg build                                # extract papers not yet in the graph, rebuild it
tensorx reading kg status

# Look things up
tensorx reading search "deep brain stimulation" --limit 10
tensorx reading get <item_key>                          # metadata, PDF path, outline, KG summary
tensorx reading collections
```

| Command | Meaning |
|---|---|
| `import` | Needs one of `--doi`, `--arxiv`, `--pdf` or `--title`. Optional `--collection <name>` (created if missing), `--tag <tag>` (repeatable), `--outline` (outline after import), `--no-download` (skip PDF download). Exits with 5 if the paper is already in the library |
| `outline` | Runs the app's AI outline flow, saves the outline and adds the paper to the knowledge graph. The paper needs a PDF. Uses account points |
| `kg build` | Extracts papers that have an outline but were never extracted, then rebuilds the global graph. `--no-extract` only rebuilds; `--model` picks the model |
| `search` | Searches titles and abstracts; prints item_key, year, title, authors |
| `get` | Everything about one paper |

Get an `item_key` from the output of `search` or `import`.

## Common options and output

| Option | Meaning |
|---|---|
| `--json` | JSON output; errors too: `{"ok": false, "error": {"code", "message"}, "exitCode"}` |
| `-q, --quiet` | No progress messages |
| `--no-launch` | Fail instead of starting an app that is not running |
| `--timeout <seconds>` | How long to wait for a job, default 1800 |

Progress always goes to stderr, so stdout stays parseable with `--json`.

| Exit code | Meaning |
|---|---|
| 0 | Success |
| 1 | General error |
| 2 | Invalid arguments |
| 3 | App or TeX runtime unavailable |
| 4 | Not found (paper, project, main file, PDF) |
| 5 | Imported paper already exists |
| 6 | Compilation failed |
| 7 | App not signed in |

## AI agents

[`skills/tensorx/SKILL.md`](skills/tensorx/SKILL.md) is an agent skill that tells an agent when and how to call `tensorx`. For Claude Code:

```bash
# macOS / Linux
mkdir -p ~/.claude/skills/tensorx && cp skills/tensorx/SKILL.md ~/.claude/skills/tensorx/
```

```powershell
# Windows PowerShell
New-Item -ItemType Directory -Force "$HOME\.claude\skills\tensorx" | Out-Null
Copy-Item skills\tensorx\SKILL.md "$HOME\.claude\skills\tensorx\"
```

Other agents that accept skills or custom instructions can use the file's contents as is.

## Security and privacy

- `tensorx` only talks to the apps on `127.0.0.1`. Each app start creates a fresh random token in the current user's app data folder; requests without it, and requests from a browser, are refused.
- The only network traffic is what the apps already do: fetching paper metadata and PDFs, AI outline and knowledge-graph extraction, and downloading missing TeX packages.

## Feedback

Please open an issue in this repository.

## License

The documentation and skill in this repository are licensed under [MIT](LICENSE).
