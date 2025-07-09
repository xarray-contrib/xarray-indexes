---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# The default `PandasIndex`

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

1. {py:class}`xarray.indexes.PandasIndex` can wrap _one dimensional_ {py:class}`pandas.Index` objects to allow indexing along 1D coordinate variables. These indexes can apply to both {term}`"dimension" coordinates <xarray:Dimension coordinate>` and {term}`"non-dimension" coordinates <xarray:Non-dimension coordinate>`.
1. When opening or constructing a new Dataset or DataArray, Xarray creates by default a {py:class}`xarray.indexes.PandasIndex` for each {term}`"dimension" coordinate <xarray:Dimension coordinate>`.
1. It is possible to either drop those default indexes or skip their creation.

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

It has created by default a {py:class}`~xarray.indexes.PandasIndex` for each of
the "lat", "lon" and "time" dimension coordinates, as we can also see below via
the {py:attr}`xarray.Dataset.xindexes` property.

```{code-cell} python
ds_air.xindexes
```

Those indexes are used under the hood for, e.g., label-based selection.

```{code-cell} python
ds_air.sel(time="2013")
```

### Set indexes for non-dimension coordinates

Xarray does not automatically create an index for non-dimension coordinates like
the "season (time)" coordinate added below.

```{code-cell} python
ds_air.coords["season"] = ds_air.time.dt.season
ds_air
```

Without an index, it is not possible select data based on the "season"
coordinate.

```{code-cell} python
---
tags: [raises-exception]
---
ds_air.sel(season="DJF")
```

However, it is possible to manually set a `PandasIndex` for that 1-dimensional
coordinate.

```{code-cell} python
ds_extra = ds_air.set_xindex("season", xr.indexes.PandasIndex)
ds_extra
```

Which now enables label-based selection.

```{code-cell} python
ds_extra.sel(season="DJF")
```

It is not yet supported to provide labels to {py:meth}`xarray.Dataset.sel` for
multiple index coordinates sharing common dimensions (unless those coordinates
also share the same index object, e.g., like shown in the {doc}`PandasMultiIndex example <pdmultiindex>`).

```{code-cell} python
---
tags: [raises-exception]
---
ds_extra.sel(season="DJF", time="2013")
```

### Drop indexes

Indexes are not always necessary and (re-)computing them may introduce some
unwanted overhead.

The code line below drops the default indexes that have been created when
opening the example dataset.

```{code-cell} python
ds_air.drop_indexes(["time", "lat", "lon"])
```

### Skip the creation of default indexes

Let's re-open the example dataset above, this time with no index.

```{code-cell} python
ds_air_no_index = xr.tutorial.open_dataset(
    "air_temperature", create_default_indexes=False
)

ds_air_no_index
```

Like {py:func}`xarray.open_dataset`, indexes are created by default for
dimension coordinates when constructing a new Dataset.

```{code-cell} python
ds = xr.Dataset(coords={"x": [1, 2], "y": [3, 4, 5]})

ds
```

Also when assigning new coordinates.

```{code-cell} python
ds.assign_coords(u=[10, 20])
```

To skip the creation of those default indexes, we need to explicitly create a
new {py:class}`xarray.Coordinates` object and pass `indexes={}` (empty
dictionary).

```{code-cell} python
coords = xr.Coordinates({"u": [10, 20]}, indexes={})

ds.assign_coords(coords)
```
