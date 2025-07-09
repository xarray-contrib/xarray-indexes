---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# Integer ranges with `pd.RangeIndex`

````{grid}
```{grid-item}
:columns: 3
```{image} https://pandas.pydata.org/docs/_static/pandas.svg
---
alt: Pandas logo
width: 200px
align: center
---
```
````

## Highlights

1. Like other pandas Index types, a {py:class}`pandas.RangeIndex` object may wrapped in an {py:class}`xarray.indexes.PandasIndex`.
1. Unlike other pandas Index types, we always want to assign a `pandas.RangeIndex` directly instead of setting it from an existing coordinate variable.
1. Xarray preserves the memory-saving `pandas.RangeIndex` structure by wrapping it in a lazy coordinate variable instead of a fully materialized array.

## Example

### Assigning

```{code-cell} python
import pandas as pd
import xarray as xr
```

```{code-cell} python
---
tags: [remove-cell]
---
%xmode minimal

xr.set_options(
    display_expand_indexes=True,
    display_expand_attrs=False,
);
```

```{code-cell} python
idx = xr.indexes.PandasIndex(pd.RangeIndex(1_000_000_000), dim="x")

ds = xr.Dataset(coords=xr.Coordinates.from_xindex(idx))
ds
```

### Lazy coordinate

The `x` coordinate variable associated with the range index is lazy (i.e., all
array values are not fully materialized in memory).

```{code-cell} python
ds.x
```

```{important}
`ds.x.values` will materialize all values in-memory! `x` may behave like a "coordinate variable bomb" 💣.
```

### Indexing

Slicing along the `x` dimension preserves the range index -- although with a new
range -- and keeps a lazy associated coordinate variable.

```{code-cell} python
sliced = ds.isel(x=slice(1_000, 50_000, 100))

sliced.x
```

```{code-cell} python
sliced.xindexes["x"]
```

Indexing with arbitrary values along the same dimension converts the underlying
pandas index type (this is all handled by pandas).

```{code-cell} python
indexed = ds.isel(x=[10, 55, 124, 265])

indexed.x
```

```{code-cell} python
indexed.xindexes["x"]
```
