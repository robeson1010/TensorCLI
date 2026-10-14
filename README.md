# TensorCLI

[English](README.md) | [简体中文](README.zh-CN.md)

`tensorx` is the command line for **TensorReading** (literature management and reading) and **TensorWriting** (LaTeX writing). You, your scripts, and any AI agent that can run terminal commands can use it to import papers, build the knowledge base (outlines and the knowledge graph), search the library and browse it by collection or tag, generate and edit papers' AI blogs, and compile LaTeX projects to PDF.

This repository holds only the download links, the usage guide and the agent skill. `tensorx` is installed together with TensorReading; there is nothing else to install.

## Download and install

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

## Requirements

- **A TensorX Pro membership.** The app involved must be signed in with a Pro (or Enterprise) account: library commands check TensorReading's sign-in, compile commands check TensorWriting's. Signed out gives exit code 7; an account that is not Pro gives exit code 8. Sign in inside the app, or run:

  ```bash
  tensorx reading login     # opens the sign-in page in your browser and waits
  tensorx writing login
  ```

- The desktop apps do the work. If an app is not running, `tensorx` starts it; with `--no-launch` it fails instead.
- TensorWriting needs TeX Live installed (from its account menu).
- TensorReading and TensorWriting can run at the same time; they use different local ports and do not conflict.

## Quick start

```bash
tensorx status                                        # are the apps running, signed in, on Pro
tensorx writing compile ./my-paper                    # compile a LaTeX project; PDF next to the main file
tensorx reading import --arxiv 1706.03762 --kg        # import a paper and add it to the knowledge base
tensorx reading search "transformer" --limit 5        # search the library
```

## Commands

| Command | What it does |
|---|---|
| `tensorx status` | Both apps: running, version, signed-in account, membership (never starts an app) |
| `tensorx skill [-o <file>]` | Print or save the agent skill (needs no app and no sign-in) |
| `tensorx reading status` \| `login` | TensorReading status and library location; sign in through the browser |
| `tensorx reading search [<query>] [--collection …] [--tag …]` | Search titles and abstracts, or list a collection's or tag's papers |
| `tensorx reading get <item_key>` | One paper: metadata, tags, PDF path, outline, knowledge-graph summary, AI blog status |
| `tensorx reading collections` | The collection tree with each collection's paper count |
| `tensorx reading tags` | Every tag with its paper count |
| `tensorx reading tag add\|remove <item_key> <tag>…` | Add or remove a paper's tags |
| `tensorx reading blog list` | Papers that have an AI blog, and whether it is published |
| `tensorx reading blog <item_key>` | Print a paper's AI blog (Markdown) |
| `tensorx reading blog generate <item_key>` | Have AI write the blog from the paper's PDF |
| `tensorx reading blog write <item_key> <file.md>` | Replace the blog's text (from a file or stdin) |
| `tensorx reading import …` | Import a paper |
| `tensorx reading kg build …` | Build the knowledge base (outline and knowledge graph together) |
| `tensorx reading kg status` | Size and build time of the knowledge graph |
| `tensorx writing status` \| `login` | TensorWriting status; sign in through the browser |
| `tensorx writing open <project-dir>` | Open a project in TensorWriting without compiling |
| `tensorx writing compile [<project-dir>\|<main.tex>]` | Compile a LaTeX project to PDF |

Run `tensorx --help` for every option.

## TensorWriting: compile a LaTeX project

```bash
tensorx writing compile                            # compile the current folder
tensorx writing compile ./my-paper                 # compile a project folder
tensorx writing compile ./my-paper/main.tex        # name the main file (its folder is the project)
tensorx writing compile ./my-paper -o out/paper.pdf
tensorx writing compile ./thesis --main chapters/main.tex --engine xelatex
tensorx writing open ./my-paper                    # open in the app without compiling
```

`compile` switches the TensorWriting window to the project, picks the main file and engine by the app's own rules, builds it with the app's compiler, and writes the PDF:

- by default next to the main file with the same name (`main.tex` → `main.pdf`);
- or wherever `-o` says. The path must end in `.pdf` and its folder must exist.

| Option | Meaning |
|---|---|
| `--main <relative path>` | Main file; same as "Set as main file" in the app, saved in the project settings |
| `--engine pdflatex\|xelatex\|lualatex` | Engine for this build only; the project setting is unchanged |
| `--mode full\|quick` | Full (default) or quick compilation |
| `-o, --output <file.pdf>` | Where to write the PDF |
| `--log` | Also print the full compile log |

A failed build lists its errors as `file:line: error: message` and exits with code 6. Missing packages are downloaded automatically; progress goes to stderr. Unsaved edits in the project are saved before compiling.

## TensorReading: papers and the knowledge base

### Import

```bash
tensorx reading import --doi 10.1038/s41586-021-03819-2
tensorx reading import --arxiv 1706.03762 --collection "Transformers" --tag to-read
tensorx reading import --pdf ./paper.pdf --title "Paper title"
tensorx reading import --pdf ./paper.pdf --kg           # add to the knowledge base right away
```

- Needs one of `--doi`, `--arxiv`, `--pdf` or `--title`. Import checks for duplicates, looks up the metadata, and downloads the arXiv or open-access PDF.
- `--collection <name>` files it there, creating the collection if needed. `--tag <tag>` is repeatable. `--url <url>` records the paper's web address.
- `--kg` adds the paper to the knowledge base after import (outline and knowledge graph). `--no-download` skips the PDF download.
- A paper already in the library is not added again: exit code 5, with the existing paper's `item_key`.

### Knowledge base (outline + knowledge graph)

Outlines and the knowledge graph go together; there is no separate outline command.

```bash
tensorx reading kg build <item_key>                     # these papers
tensorx reading kg build <key1> <key2> --force          # rebuild even if already in it
tensorx reading kg build --all                          # every paper not in the knowledge base yet
tensorx reading kg build --no-extract                   # only rebuild the graph, no AI calls
tensorx reading kg status
```

For each paper:

- no outline yet: the outline and the knowledge graph are generated together from its PDF (the paper needs a PDF);
- an outline but no extraction: only the knowledge-graph extraction runs;
- already in the knowledge base: skipped unless `--force`.

The global knowledge graph is rebuilt at the end. Name the papers or pass `--all`. Building the knowledge base uses AI services and spends account points; on a large library `--all` can cover many papers, so check first. `--model <model>` picks the extraction model.

### Look things up

```bash
tensorx reading search "deep brain stimulation" --limit 10
tensorx reading search --collection "LFP"                 # every paper in a collection
tensorx reading search --tag "deep learning" --tag LFP    # every paper carrying both tags
tensorx reading search "transformer" --collection "LLM"   # keywords within a collection
tensorx reading get <item_key>
tensorx reading collections
tensorx reading tags
```

- `search` looks in titles and abstracts and prints item_key, year, title, authors. With `--json` the items are CSL-JSON, ready for building references.
- Keyword searches show 20 results by default; with only `--collection` / `--tag` you get them all. `--limit N` changes it, `--limit 0` means no limit. When results are cut off, the output says how many there are in total.
- `--collection` takes the collection's name (case-insensitive). If several collections share a name, the error lists their ids; pass the id instead. Papers in subcollections are not included.
- Several `--tag`s mean the paper must carry all of them. Tag names ignore letter case.
- `get` returns metadata, authors, tags, collections, the PDF's absolute path, the outline, a knowledge-graph summary (entity and relation counts, one-line summary), and the AI blog's status.
- `collections` and `tags` show how many papers each has. Papers in the recycle bin are never counted or listed.

Get an `item_key` from the output of `search` or `import`.

### Tags

```bash
tensorx reading tag add <item_key> "to read" "deep learning"
tensorx reading tag remove <item_key> "to read"
```

- `add` reuses an existing tag that differs only in letter case, instead of creating a near-duplicate.
- `remove` exits with code 4 when the paper had none of the tags named.
- The library view in the app refreshes by itself.

### AI blog

Each paper can have one AI blog: an article written from the paper, with figures cropped from the PDF. It is the same blog as the app's AI blog tool.

```bash
tensorx reading blog list                          # papers with a blog: published or not
tensorx reading blog <item_key>                    # print it (Markdown with YAML frontmatter)
tensorx reading blog generate <item_key>           # AI writes it from the paper's PDF
tensorx reading blog generate <item_key> --force   # write it again, replacing the current one
tensorx reading blog write <item_key> edited.md    # replace its text; `-` reads stdin
```

- **Publishing stays in the app.** `tensorx` never publishes to tensorx.xin; open the AI blog tool and click Publish. `blog list` and `get` show whether a blog is published and its address.
- `generate` uses AI services and spends account points, takes a few minutes, and needs the paper's PDF. On a paper that already has a blog it stops with exit code 5 unless you pass `--force`.
- `write` keeps the current frontmatter when the file has none, and always keeps the published post's id and address, so publishing again updates the same post. If the blog is open in the app, the editor reloads it.
- A blog file with nothing after its frontmatter (left by opening a paper in the editor) counts as no blog.

## Common options and output

| Option | Meaning |
|---|---|
| `--json` | JSON output; errors too: `{"ok": false, "error": {"code", "message"}, "exitCode"}` |
| `-q, --quiet` | No progress messages |
| `--no-launch` | Fail instead of starting an app that is not running |
| `--timeout <seconds>` | How long to wait for a job, default 1800. The job keeps running in the app after a timeout |

Progress always goes to stderr, so stdout stays parseable with `--json`. In a terminal, long jobs (import, knowledge base, AI blog) show their steps like the app does: finished steps get a tick and their time, the current one updates in place, with a bar when its size is known. When stderr is not a terminal, only finished steps are printed, one line each. The [skill](skills/tensorx/SKILL.md) documents each command's JSON fields.

| Exit code | Meaning |
|---|---|
| 0 | Success |
| 1 | General error (including timeouts and knowledge-base papers that failed) |
| 2 | Invalid arguments |
| 3 | App or TeX runtime unavailable |
| 4 | Not found (paper, project, main file, PDF) |
| 5 | Already exists (an imported paper, or a blog `generate` would replace) |
| 6 | Compilation failed |
| 7 | App not signed in |
| 8 | Account is not on a Pro membership |

## AI agents

Any AI agent that can run terminal commands can use `tensorx`. [`skills/tensorx/SKILL.md`](skills/tensorx/SKILL.md) is an agent skill that tells it when and how to call `tensorx` and how to read its JSON output and exit codes; `tensorx` carries the same file:

```bash
tensorx skill                                      # print the skill
tensorx skill -o <skills folder>/tensorx/SKILL.md  # save it (folders are created)
```

**Let the agent install it (recommended)**: send your agent this message; it knows where it loads skills from.

> Please install tensorx as one of your skills:
> 1. Run `tensorx skill` and read the guide;
> 2. Run `tensorx skill -o <the folder you load skills from>/tensorx/SKILL.md` to save it. If you do not support skills, add the guide to the project instructions file you read (for example AGENTS.md);
> 3. Run `tensorx status --json` to check that TensorReading and TensorWriting can be reached;
> 4. Tell me where you saved the skill and whether a new session is needed for it to take effect.

**Install it yourself**: put SKILL.md in the agent's skills folder (usually `skills/tensorx/SKILL.md`), or paste it into the instructions file or custom instructions it reads. The TensorCLI page in TensorReading's toolbox also has save and copy buttons.

Then just ask your agent, for example:

- Compile ./my-paper with tensorx; fix any errors and compile again until it produces a PDF
- Import arXiv 2404.03425 into TensorReading, put it in the "To read" collection, and add it to the knowledge base
- Find papers on remote sensing foundation models in my TensorReading library and add citations to refs.bib
- List every paper in my "LFP" collection that has no AI blog yet, and generate one for the three most recent
- Read the AI blog of <item_key>, tighten the introduction and fix the figure captions, then save it back

## Troubleshooting

| Symptom | What to do |
|---|---|
| The shell cannot find `tensorx` | Start TensorReading once, then open a new terminal; on macOS / Linux make sure `~/.local/bin` is on your PATH |
| Exit code 7 (not signed in) | Sign in to your TensorX account in the app, or run `tensorx reading login` / `tensorx writing login` |
| Exit code 8 (not Pro) | `tensorx` is available to Pro and Enterprise members |
| Exit code 3, TeX Live not installed | Install TeX Live from TensorWriting's account menu |
| "No response after starting" the app | The app is too old for `tensorx`; update it |
| A command fails with "HTTP 404" | The app is older than this `tensorx` command; update TensorReading |
| Timeout | A first compile downloads packages and knowledge-base jobs take a while; raise `--timeout`, the job finishes in the app either way |

## Security and privacy

- `tensorx` only talks to the apps on `127.0.0.1`. Each app start creates a fresh random token in the current user's app data folder; requests without it, and requests from a browser, are refused.
- The only network traffic is what the apps already do: fetching paper metadata and PDFs, the knowledge base's AI extraction, AI blog generation, downloading missing TeX packages, and checking the membership with TensorX. Publishing a blog happens only when you click Publish in the app.

## Feedback

Please open an issue in this repository.

## License

The documentation and skill in this repository are licensed under [MIT](LICENSE).
