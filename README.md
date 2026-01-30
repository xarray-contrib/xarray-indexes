# xarray-indexes

A gallery of custom Xarray indexes

## Building the documentation

```bash
uv run sphinx-build docs docs/_build/html
```

Then open `docs/_build/html/index.html` in your browser. For live reload during development:

```bash
uv run sphinx-autobuild docs docs/_build/html
```

## Editing documentation notebooks

The notebooks in `docs/blocks/` are paired with Markdown files using [jupytext](https://jupytext.readthedocs.io/). This means you can edit either the `.ipynb` or `.md` file—just sync them before committing.

### Essential commands

```bash
# Sync a notebook with its markdown (run this before committing)
uvx jupytext --sync docs/blocks/ndpoint.ipynb

# Sync all notebooks
uvx jupytext --sync docs/blocks/*.ipynb
```

### Tips

- Always commit both `.md` and `.ipynb` files together
- Run `jupytext --sync` before committing to ensure files match
- The `.md` files use MyST format with `{code-cell}` blocks for executable code
