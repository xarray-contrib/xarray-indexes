---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# Stack and unstack with `PandasMultiIndex`

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

1. An {py:class}`xarray.indexes.PandasMultiIndex` is associated with multiple coordinate variables sharing the same dimension.
1. Create PandasMultiIndex from PandasIndex using {py:meth}`xarray.Dataset.stack` and convert back with {py:meth}`xarray.Dataset.unstack`.
1. Labels of coordinates associated with a PandasMultiIndex can be passed all at once to `.sel`.

## Example

Let's open a tutorial dataset.

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

```{code-cell} python
ds_air = xr.tutorial.open_dataset("air_temperature")
ds_air
```

### Stack / Unstack

Stacking the "lat" and "lon" dimensions of the example dataset results here in
the corresponding "lat" and "lon" stacked coordinates both associated with a
`PandasMultiIndex` by default.
The underlying data are _reshaped_ to collapse the `lat` and `lon` dimensions to a new `space` dimension.

```{code-cell} python
stacked = ds_air.stack(space=("lat", "lon"))
stacked
```

The multi-index allows retrieving the original, unstacked dataset where the
"lat" and "lon" dimension coordinates have their own `PandasIndex`.

```{code-cell} python
unstacked = stacked.unstack("space")
unstacked
```

### Assigning

We can also directly associate a {py:class}`~xarray.indexes.PandasMultiIndex`
with existing coordinates sharing the same dimension.

```{code-cell} python
ds_air = (
    ds_air
    .assign_coords(season=ds_air.time.dt.season)
    .rename_vars(time="datetime")
    .drop_indexes("datetime")
)

ds_air
```

```{code-cell} python
multi_indexed = ds_air.set_xindex(["season", "datetime"], xr.indexes.PandasMultiIndex)
multi_indexed
```

### Indexing

Contrary to what is shown in {doc}`the default PandasIndex <pdindex>` example,
it is here possible to provide labels to {py:meth}`xarray.Dataset.sel` for both
of the multi-index time coordinates.

```{code-cell} python
multi_indexed.sel(season="DJF", datetime="2013")
```

Chaining `.sel` calls for those coordinates each with their own index would
yield equivalent results, though.

```{code-cell} python
single_indexed = ds_air.set_xindex("datetime").set_xindex("season")

single_indexed.sel(season="DJF").sel(datetime="2013")
```

### Assigning a `pandas.MultiIndex`

It is easy to wrap an existing {py:class}`pandas.MultiIndex` object into a new Xarray
Dataset or DataArray.

```{code-cell} python
import pandas as pd

midx = pd.MultiIndex.from_product([["a", "b"], [1, 2]], names=("foo", "bar"))
midx
```

This can be done via {py:meth}`xarray.Coordinates.from_pandas_multiindex`.

```{code-cell} python
midx_coords = xr.Coordinates.from_pandas_multiindex(midx, dim="x")

ds = xr.Dataset(coords=midx_coords)
ds
```
