---
name: marimo-library
description: Where code lives in a marimo project that has a local Python library, notebook cell or library module, and how to work on both while a notebook is running. Use when starting a marimo project, when a repo has marimo notebooks next to a local package, or when moving code out of a notebook.
license: Apache-2.0
compatibility: Needs uv. Work on a running notebook needs the marimo-pair skill (https://github.com/marimo-team/marimo-pair).
---

# Marimo + library

A marimo project is a uv project with two zones:

- **Notebooks**: marimo `.py` files. They hold what belongs to one notebook: wiring, UI state, and code still being explored.
- **Library**: a normal Python package that the notebooks import. It has tests, docstrings, linting and formatting.

Code starts in cells and **graduates** into the library. The library exists from day one, even when empty, so graduating is a small move and never a migration.

There are two kinds of notebook. The rules are the same; the end state differs:

- **Exploration**: an analysis. Cells may be rough, and they graduate as the rules below fire.
- **App**: the notebook is the product, served with `marimo run`. Each cell is a call into the library plus the wiring between widgets, and new logic is written in the library first. Test it through the browser: see the `marimo-playwright` skill.

In an existing project, its layout and its own docs win over the defaults here.

## Graduation rules

Move code from a cell into the library when any of these is true:

1. **Reuse**: a second notebook, a test, or a script wants it.
2. **Stability**: it is no longer edited; it just runs.
3. **Quality**: it deserves a test, a docstring, or lint attention.
4. **Re-reading**: someone keeps reading it only to work on something downstream.

To graduate, turn the cell body into a library function with a docstring, and replace the cell with a call such as `df = load_and_clean(path)`. Add a test if rule 3 fired. Then run the library tests and re-run the affected cells to confirm the behaviour is unchanged.

If you had to read upstream cells to do a downstream task, rule 4 fired on you: tell the user that code is a candidate.

Slow or one-shot work (model calls, big downloads) goes in a script that writes files the notebook reads, so opening the notebook stays fast.

## Library shape

- Paths, data and settings reach library functions as arguments, passed in by the notebook.
- Code that imports marimo (widgets, layouts) lives in its own module, apart from the logic. The logic then tests without marimo, and several notebooks share the widgets.

## Running notebooks

The marimo-pair skill owns every interaction with a running notebook: finding it, running code in it, changing its cells. Load it first. While a notebook runs, its `.py` file belongs to the kernel, so cell changes go through marimo-pair. This holds under `--watch` too: that flag is for picking up `git pull` and `git checkout`. A notebook that is not running is a normal file.

This skill adds:

- **Library, tests and scripts are always yours to edit** with normal file tools, whether or not a notebook is running.
- **A library edit leaves cells stale.** With `auto_reload = "lazy"` (see New project), marimo marks every cell that imports the changed module as stale, so the kernel never silently runs old code. Run the stale cells that only compute. Leave the cells that create `mo.state` to the user: re-running them resets the user's UI.
- **Map, then read.** Print one line per cell, then read the code of only the cells you will change:

  ```python
  import builtins
  import marimo._code_mode as cm

  async with cm.get_context() as ctx:
      for cid in ctx.cells.keys():
          impl = ctx.graph.cells[cid]
          refs = sorted(r for r in impl.refs if not hasattr(builtins, r))
          print(cid, sorted(impl.defs), "<-", refs)
  ```

  Then `print(ctx.cells["<id>"].code)` for those cells. `cm` is a private API that changes between marimo versions: when the map fails, repair it from `help(cm)`. Every cell you read stays in your context for the rest of the session.
- **Probe live state.** To learn the shape of `df`, run `df.head()` or `df.dtypes` in the scratchpad. That answers faster than the cells that built it.

## New project

```bash
uv init --lib <name>
cd <name>
uv add marimo
mkdir notebooks
```

`uv init --lib` creates `src/<package>/` with a build backend, so uv installs the package in editable mode and `import <package>` works from any folder. Notebooks go in `notebooks/`, tests in `tests/`.

Add to `pyproject.toml`:

```toml
[tool.marimo.runtime]
auto_reload = "lazy"

# Only with ruff: marimo formats a notebook itself on every save, at its own width.
[tool.ruff.format]
exclude = ["notebooks/*.py"]
```

Add `__marimo__/` to `.gitignore`.

Dependencies go in `pyproject.toml` through `uv add`. An inline `# /// script` header in a notebook is for a single-file notebook opened with `--sandbox`; marimo ignores it otherwise.
