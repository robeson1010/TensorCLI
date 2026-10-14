---
name: tensorx
description: Drive the user's TensorReading literature library and TensorWriting LaTeX editor through the `tensorx` command line. Use to compile a LaTeX project to PDF and read its errors, to import papers (DOI, arXiv, local PDF), build or inspect the knowledge base (outline and knowledge graph), search and read papers in the user's library, list a collection's or tag's papers, add or remove tags, or generate, read and edit a paper's AI blog.
---

# tensorx

`tensorx` ships with TensorReading and talks to the two desktop apps on this machine. The apps do the work: compiling happens in TensorWriting, imports and the knowledge base in TensorReading. If an app is not running, `tensorx` starts it and waits.

## Rules

- Always pass `--json` and parse stdout. Progress lines go to stderr; ignore them.
- Every JSON answer has `ok`. On failure: `{"ok": false, "error": {"code", "message"}, "exitCode"}`.
- The app must be signed in to a TensorX account on a **Pro** (or Enterprise) membership. On `AUTH_REQUIRED` (exit 7), tell the user to sign in in the app (or run `tensorx reading login` / `tensorx writing login`, which opens the browser and waits). On `PRO_REQUIRED` (exit 8), tell the user tensorx needs a Pro membership. Do not retry in a loop.
- `kg build`, `import --kg` and `blog generate` call paid AI services on the user's account. Run them only when the user asked for the knowledge base (or outlines, which are part of it) or for the AI blog.
- Publishing a blog to tensorx.xin is the user's click in the app. `tensorx` never publishes; after changing a published blog, remind the user to publish again.
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
tensorx reading search --tag "<tag>" [--tag ...] --json  # every paper carrying all these tags
tensorx reading search --collection "<name or id>" --json # every paper directly in that collection
tensorx reading get <item_key> --json                   # item, creators, tags, collections, pdf, folder, outline, knowledge, blog
tensorx reading collections --json                      # collections[]: id, name, parent_id, itemCount
tensorx reading tags --json                             # tags[]: id, name, count
tensorx reading tag add <item_key> "<tag>" [...] --json
tensorx reading tag remove <item_key> "<tag>" [...] --json
```

- `search` returns CSL-JSON (`id`, `title`, `author[]`, `issued`, `container-title`, `DOI`, …). Build BibTeX entries from it when adding citations to a LaTeX project.
- `get` returns `pdf` (absolute path or null). Read the PDF from that path when you need the full text. `outline` is `{"sections":[{title, level, page, summary, children}]}` or null. `knowledge` summarizes the paper's knowledge-graph extraction (`entities`, `relations`, `oneLineSummary`, `keyFindings`) or is null.
- `search` also returns `total` (matches before `--limit`). Keyword searches default to 20 results; with no keywords and a `--collection`/`--tag`, all matches are returned. `--limit 0` means no limit. Filters combine (keywords AND collection AND every tag). Recycle-bin papers never appear.
- `--collection` takes a name (case-insensitive) or, when several collections share a name (error lists their ids), an id. Subcollections are not included. Unknown collection or tag exits 4.
- Tag names match ignoring ASCII case. `search --tag` with a tag that does not exist exits 4. `itemCount` and tag `count` leave out the recycle bin.
- `tag add|remove` result: `tags[]` (the paper's tags afterwards), `added[]`, `removed[]`, `missing[]` (names to remove the paper did not carry). `add` reuses an existing tag that differs only in case. `remove` exits 4 when nothing was removed. The app's library view refreshes by itself.

## AI blog

Each paper can have one AI blog: `blog_post.md` (YAML frontmatter + Markdown) in the paper folder, figures in `figs/` next to it. Publishing to tensorx.xin is done by the user in the app (the 发布 button); the CLI never publishes.

```bash
tensorx reading blog list --json                          # blogs[]: itemKey, paperTitle, title, published, postUrl, chars, figures, modifiedAt, path
tensorx reading blog <item_key> --json                    # blog: the same fields + markdown
tensorx reading blog generate <item_key> [--force] --json # AI writes it from the PDF (takes a few minutes)
tensorx reading blog write <item_key> <file.md> --json    # replace it; `-` reads stdin
```

- `get` also returns `blog` (same fields, no markdown) or null. `published` is true once the app has published it (`postUrl` set).
- A blog with nothing past its frontmatter counts as no blog: it is not listed, `get` gives `blog: null`, `blog <key>` exits 4, and `generate` writes it without `--force`.
- `generate` calls a paid AI service on the user's account; run it only when the user asked for the blog. It exits 5 (`DUPLICATE`) when the paper already has a blog; add `--force` only after the user agrees to overwrite it. Exit 4 when the paper has no PDF. Result: `figures`, `failedFigures[]`, `equations`, `chars`, `replaced`, `published`, `postUrl`.
- To edit: `blog <key> --json`, change `markdown`, save it to a file, `blog write <key> <file>`. Keep image references as `figs/<name>.png`; the frontmatter `cover` names one of those files. Text without frontmatter keeps the current frontmatter. `post_id`/`post_url` are always kept so the next publish updates the same post.
- After changing a published blog, tell the user to publish again in the app; the site does not change until then.

## Import

```bash
tensorx reading import --doi <doi> --json
tensorx reading import --arxiv <id> --json
tensorx reading import --pdf <file.pdf> [--title "<title>"] --json
#   optional: --collection "<name>" (created if missing) --tag <tag> (repeatable) --kg --no-download
```

- Result: `itemId`, `itemKey`, `title`, `status` (`created` | `duplicate`), `pdf`, `pdfSource`.
- Exit code 5 with `status: "duplicate"`: the paper was already in the library; `itemKey` is the existing paper. Treat it as found, not as an error.
- `pdf: null` means no open-access PDF was found; ask the user for a local file and use `--pdf`.
- `--kg` also adds the paper to the knowledge base; the result then has `knowledge` (same shape as `kg build`).

## Knowledge base (outline + knowledge graph)

The outline and the knowledge-graph extraction are one step; there is no separate outline command.

```bash
tensorx reading kg build <item_key> [<item_key> ...] [--force] --json   # these papers
tensorx reading kg build --all [--force] --json                         # every paper not in the knowledge base yet
tensorx reading kg build --no-extract --json                            # only rebuild the graph, no AI calls
tensorx reading kg status --json
```

- For each paper: without an outline, the outline and extraction are generated together from its PDF; with an outline but no extraction, only the extraction runs. Papers already in the knowledge base are skipped unless `--force`.
- Result: `papers[]` (`itemKey`, `title`, `status` = built | extracted | upToDate | noPdf | failed, `sections`, `error`), counts `built`, `extracted`, `upToDate`, `noPdf`, `failed`, and `graph`. Exit 1 when a paper failed (or a named paper has no PDF); the others are saved.
- `--all` can mean many AI calls on a large library; confirm with the user first.
- Read a paper's outline afterwards with `tensorx reading get <item_key>`.
- `kg status`: `extractions` and `graph` (`papers`, `nodes`, `edges`, `clusters`, `builtAt`).

## Exit codes

| Code | Meaning | What to do |
|---|---|---|
| 0 | Success | |
| 1 | General error, timeout | Report the message |
| 2 | Bad arguments | Fix the command |
| 3 | App or TeX runtime unavailable | Ask the user to install or start the app / install TeX Live in TensorWriting |
| 4 | Not found (paper, project, main file, PDF) | Check the key or path |
| 5 | Already exists (import duplicate, or `blog generate` on a paper that has a blog) | Import: use the returned `itemKey`. Blog: ask before `--force` |
| 6 | Compile failed | Fix the sources from `diagnostics` |
| 7 | Not signed in | Ask the user to sign in |
| 8 | Account is not Pro | Tell the user tensorx needs a Pro membership |

Long jobs (first compile with package downloads, knowledge-base builds) can take minutes; the default wait is 30 minutes, change it with `--timeout <seconds>`.
