---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# Stick coordinates together with `PandasMultiIndex`

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

1. An {py:class}`xarray.indexes.PandasMultiIndex` is associated with multiple coordinates sharing the same dimension.
1. It permits using `.sel` with labels given for several of those coordinates.
1. It is the index used by default for `.stack` and `.unstack`.

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

### Assigning

We need multiple coordinates sharing the same dimension.

```{code-cell} python
ds_air = (
    ds_air
    .assign_coords(season=ds_air.time.dt.season)
    .rename_vars(time="datetime")
    .drop_indexes("datetime")
)

ds_air
```

Now we can assign a {py:class}`~xarray.indexes.PandasMultiIndex` to the time
coordinates.

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

### Stack / Unstack

### Create coordinates from a `pandas.MultiIndex`

It is easy to wrap an existing `pandas.MultiIndex` object into a new Xarray
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
