---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# Floating point ranges with `RangeIndex`

## Highlights

1. Pandas has no equivalent of {py:class}`pandas.RangeIndex` for floating point
   ranges. Fortunately, there is {py:class}`xarray.indexes.RangeIndex` that
   works with real numbers.
1. Xarray's `RangeIndex` is built on top of
   {py:class}`xarray.indexes.CoordinateTransformIndex` and therefore supports
   very large ranges represented as lazy coordinate variables.

## Example

### Assigning

```{code-cell} python
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

Using {py:meth}`xarray.indexes.RangeIndex.arange`.

```{code-cell} python
idx1 = xr.indexes.RangeIndex.arange(0.0, 1000.0, 1e-3, dim="x")

ds1 = xr.Dataset(coords=xr.Coordinates.from_xindex(idx1))
ds1
```

Using {py:meth}`xarray.indexes.RangeIndex.linspace`.

```{code-cell} python
idx2 = xr.indexes.RangeIndex.linspace(0.0, 1000.0, 1_000_000, dim="x")

ds2 = xr.Dataset(coords=xr.Coordinates.from_xindex(idx2))
ds2
```

### Lazy coordinate

The `x` coordinate variable associated with the range index is lazy (i.e., all
array values are not fully materialized in memory).

```{code-cell} python
ds1.x
```

```{important}
`ds.x.values` will materialize all values in-memory! `x` may behave like a "coordinate variable bomb" 💣.
```

### Indexing

Slicing along the `x` dimension preserves the range index -- although with a new
range -- and keeps a lazy associated coordinate variable.

```{code-cell} python
sliced = ds1.isel(x=slice(1_000, 50_000, 100))

sliced.x
```
