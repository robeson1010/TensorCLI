---
name: tensorx
description: Drive the user's TensorReading literature library and TensorWriting LaTeX editor through the `tensorx` command line. Use to compile a LaTeX project to PDF and read its errors, to import papers (DOI, arXiv, local PDF), generate a paper's outline, build or inspect the knowledge graph, or search and read papers in the user's library.
---

# tensorx

`tensorx` ships with TensorReading and talks to the two desktop apps on this machine. The apps do the work: compiling happens in TensorWriting, imports and AI outlines in TensorReading. If an app is not running, `tensorx` starts it and waits.

## Rules

- Always pass `--json` and parse stdout. Progress lines go to stderr; ignore them.
- Every JSON answer has `ok`. On failure: `{"ok": false, "error": {"code", "message"}, "exitCode"}`.
- The app must be signed in to a TensorX account on a **Pro** (or Enterprise) membership. On `AUTH_REQUIRED` (exit 7), tell the user to sign in in the app (or run `tensorx reading login` / `tensorx writing login`, which opens the browser and waits). On `PRO_REQUIRED` (exit 8), tell the user tensorx needs a Pro membership. Do not retry in a loop.
- `outline`, `import --outline` and `kg build` call paid AI services on the user's account. Run them only when the user asked for outlines or the knowledge graph.
- `import` adds to the user's library. Import only what the user asked for.
- Never read or write the apps' data folders directly; go through `tensorx`.

## Compile LaTeX (TensorWriting)

```bash
tensorx writing compile <project-dir | main.tex> --json [--main <rel/path.tex>] [--engine pdflatex|xelatex|lualatex] [--mode full|quick] [-o <abs-or-rel.pdf>] [--log]
```

- Default output: `<main file stem>.pdf` next to the main file. `-o` must end in `.pdf` and its folder must exist.
- Result fields: `success`, `pdf` (absolute path, when it succeeded), `pdfBytes`, `engine`, `mode`, `passes`, `elapsedMs`, `mainFile`, `diagnostics[]` (`file`, `line`, `column`, `severity` = error|warning|info, `message`), `log`.
- Exit code 6 means the document did not compile: fix the sources using `diagnostics` (errors first), then compile again. Use `--log` only when the diagnostics are not enough.
- `--engine` affects this build only. `--main` changes the project's main file in the app.
- `tensorx writing open <dir> --json` opens a project in the app without compiling.

Edit-compile loop:

1. Edit the `.tex` / `.bib` files on disk (the app reloads them).
2. `tensorx writing compile <dir> --json`.
3. If `success` is false, read `diagnostics` with `severity == "error"`, fix, repeat.

## Library (TensorReading)

```bash
tensorx reading search "<query>" --limit 10 --json      # items[]: CSL-JSON, id = item_key
tensorx reading get <item_key> --json                   # item, creators, tags, collections, pdf, folder, outline, knowledge
tensorx reading collections --json
```

- `search` returns CSL-JSON (`id`, `title`, `author[]`, `issued`, `container-title`, `DOI`, …). Build BibTeX entries from it when adding citations to a LaTeX project.
- `get` returns `pdf` (absolute path or null). Read the PDF from that path when you need the full text. `outline` is `{"sections":[{title, level, page, summary, children}]}` or null. `knowledge` summarizes the paper's knowledge-graph extraction (`entities`, `relations`, `oneLineSummary`, `keyFindings`) or is null.

## Import

```bash
tensorx reading import --doi <doi> --json
tensorx reading import --arxiv <id> --json
tensorx reading import --pdf <file.pdf> [--title "<title>"] --json
#   optional: --collection "<name>" (created if missing) --tag <tag> (repeatable) --outline --no-download
```

- Result: `itemId`, `itemKey`, `title`, `status` (`created` | `duplicate`), `pdf`, `pdfSource`.
- Exit code 5 with `status: "duplicate"`: the paper was already in the library; `itemKey` is the existing paper. Treat it as found, not as an error.
- `pdf: null` means no open-access PDF was found; ask the user for a local file and use `--pdf`.

## Outline and knowledge graph

```bash
tensorx reading outline <item_key> [<item_key> ...] [--force] --json
tensorx reading kg build [--no-extract] [--model <model>] --json
tensorx reading kg status --json
```

- `outline` needs a PDF on the paper (error code `NO_PDF`, exit 4). It saves the outline and adds the paper to the knowledge graph. Result: `outline`, `kgSaved`, `kgError`, `skipped` (true when both already existed; `--force` regenerates).
- `kg build` extracts papers that have an outline but were never extracted, then rebuilds the graph. Result: `papersWithOutline`, `upToDate`, `stale` (papers to extract), `extracted`, `failed[]`, `graph`. Exit 1 when some papers failed; the rest are saved.
- `kg status`: `extractions` and `graph` (`papers`, `nodes`, `edges`, `clusters`, `model`, `builtAt`).

## Exit codes

| Code | Meaning | What to do |
|---|---|---|
| 0 | Success | |
| 1 | General error, timeout | Report the message |
| 2 | Bad arguments | Fix the command |
| 3 | App or TeX runtime unavailable | Ask the user to install or start the app / install TeX Live in TensorWriting |
| 4 | Not found (paper, project, main file, PDF) | Check the key or path |
| 5 | Import duplicate | Use the returned `itemKey` |
| 6 | Compile failed | Fix the sources from `diagnostics` |
| 7 | Not signed in | Ask the user to sign in |
| 8 | Account is not Pro | Tell the user tensorx needs a Pro membership |

Long jobs (first compile with package downloads, outlines) can take minutes; the default wait is 30 minutes, change it with `--timeout <seconds>`.
