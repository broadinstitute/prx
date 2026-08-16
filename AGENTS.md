# AGENTS.md - prx

Project-specific guidance for agents working in this repository.
This is the public, runnable catalog of marimo notebooks for PROSPECT chemical-genetics analysis.
Planning, progress, and dated artifacts live in the sibling private repo `../prx-dev/`; cross-instance coordination lives in the primary [`jx`](https://github.com/broadinstitute/jx) repo.

`README.md` is the human entry point.
This catalog uses the shared [vignette-catalog-skills](https://github.com/carpenter-singh-lab/vignette-catalog-skills), with `vignette-catalog-compose-notebook` handling setup, execution, and composition; its specifics live in `catalog.toml`.
The skills are recorded in the tracked `skills-lock.json` but not vendored; restore them with the exact commands in `README.md` after cloning.

## Launching notebooks

Always use `--sandbox` so the PEP 723 inline metadata is provisioned:

```bash
uvx marimo edit --sandbox notebooks/nbNN_*.py
```

Do not improvise alternative launch commands.
`--sandbox` is what makes `uvx marimo` read each notebook's `/// script` dependency block; without it every notebook fails with `ModuleNotFoundError`.

## Validation Rule

After composing or editing any notebook in `notebooks/`, launch it in a marimo sandbox kernel and run all cells before reporting the task complete.
`ruff check` and `marimo check` only catch static problems; they do not catch wrong outputs, NaN-filled tables, broken altair encodings, empty plots, or silently-zero pivots.
Show the user the live URL when working interactively.

Minimal launch:

```bash
PORT=$(python -c "import socket; s=socket.socket(); s.bind(('127.0.0.1',0)); print(s.getsockname()[1])")
env -u PYTHONPATH uvx marimo edit --sandbox --headless --no-token --port $PORT notebooks/nbNN_*.py
```

Then run the installed skill's final gate:

```bash
VALIDATE=$(ls .agents/skills/vignette-catalog-compose-notebook/scripts/validate-notebook.sh .claude/skills/vignette-catalog-compose-notebook/scripts/validate-notebook.sh 2>/dev/null | head -1)
bash "$VALIDATE" notebooks/nbNN_*.py
```

The validator runs stable static checks, formatting, cold execution, and refreshes the molab session snapshot last.
Session snapshots store a `code_hash` per cell, and molab attaches the stored output only when the snapshot hash matches the source cell.
Run it after the final source edit, and commit regenerated `.json` files with changed notebooks.

## Architecture

- Catalog over library.
  Helpers live as `@app.function` cells in numbered notebooks.
  Later notebooks import from earlier notebooks by adding `notebooks/` to `sys.path`.
- Notebook deps are PEP 723 inline headers; `uv` provisions a per-notebook sandbox venv.
- `uv` is the runtime contract for users.
  Do not add Nix unless there is a concrete reason.
- Smoke-test PEP 723 headers with `uv run --no-project --python 3.13 --with ...` before launching marimo, especially when importing helpers across notebooks.
- Raw data is never edited.
  Pulled artifacts go to `data/raw/` or `data/external/<source>/`; transformations land in `data/interim/` or `data/processed/<analysis-name>/`.
  Pin SHA-256 on every `pooch` fetch.
- Do not add a Python package until repeated cross-notebook imports make the notebook-as-library pattern painful.

## Conventions

- Notebooks: `nbNN_<short_description>.py`, two-digit zero-padded.
- Top-level variable names must be unique across cells.
  Use `_chart`, `_summary`, and similar names for cell-private locals; rename values that are consumed downstream.
- Do not call `.to_pandas()` for altair; pass polars frames directly via narwhals.
- Use Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`, `chore:`).
- Use ASCII-only glyphs in code, comments, and docs.
- Prose in `.md` files uses semantic line breaks: one sentence per line, no hard wrapping at a column count.
  Markdown collapses single newlines inside a paragraph, so the rendered output is unchanged, but diffs stay local to the edited sentence instead of re-flowing every line below it.
  Applies to `AGENTS.md`, `.claude/skills/**/SKILL.md`, and any other prose-heavy markdown we revise often.

## When the Question Fits the Catalog

Almost every PROSPECT request should compose existing helpers:

- orientation and public resources -> `nb01_orientation`
- Figshare pull and reference-set spine -> `nb02_figshare_pull`
- sGR GCT parsing and strain correlation -> `nb03_hypomorph_correlation`
- structure-only and CGI-profile baselines -> `nb04_pretrained_baseline`
- same-MOA CGI signal after chemistry control -> `nb05_collapse_diagnostic`
- PCL rarefaction and CGI-shape diversity -> `nb06_cgi_shape_diversity`

Read the installed `vignette-catalog-compose-notebook` skill (and `catalog.toml`'s `[[vignette]]` table) before writing new analysis code.
