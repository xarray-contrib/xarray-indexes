---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# Nearest neighbors with `NDPointIndex`

## Highlights

1. {py:class}`xarray.indexes.NDPointIndex` is useful for dealing with
   n-dimensional coordinate variables representing irregular data.
1. It enables point-wise (nearest-neighbors) data selection using Xarray's
   [advanced indexing](https://docs.xarray.dev/en/latest/user-guide/indexing.html#more-advanced-indexing)
   capabilities.
1. By default, a {py:class}`scipy.spatial.KDTree` is used under the hood for
   fast lookup of point data. Although experimental, it is possible to plug in
   alternative structures to `NDPointIndex` (See {ref}`advanced`).

## Basic Example: Default KDTree

Let's create a dataset with random points.

```{code-cell} python
import numpy as np
import matplotlib.pyplot as plt
import xarray as xr
```

```{code-cell} python
---
tags: [remove-cell]
---
%xmode minimal
xr.set_options(
    display_expand_indexes=True,
    display_expand_data=False,
);
```

```{code-cell} python
shape = (5, 10)
xx = xr.DataArray(np.random.uniform(0, 10, size=shape), dims=("y", "x"))
yy = xr.DataArray(np.random.uniform(0, 5, size=shape), dims=("y", "x"))
data = (xx - 5)**2 + (yy - 2.5)**2

ds = xr.Dataset(data_vars={"data": data}, coords={"xx": xx, "yy": yy})
ds
```

```{code-cell} python
ds.plot.scatter(x="xx", y="yy", hue="data");
```

### Assigning

```{code-cell} python
ds_index = ds.set_xindex(("xx", "yy"), xr.indexes.NDPointIndex)
ds_index
```

### Point-wise indexing

Select one value.

```{code-cell} python
ds_index.sel(xx=3.4, yy=4.2, method="nearest")
```

Select multiple points in the `x`/`y` dimension space, using
{py:class}`xarray.DataArray` objects as input labels.

```{code-cell} python
# create a regular grid as query points
ds_grid = xr.Dataset(coords={"x": range(10), "y": range(5)})

# selection supports broadcasting of the input labels
ds_selection = ds_index.sel(
    xx=ds_grid.x, yy=ds_grid.y, method="nearest"
)

# assign selection results to the grid
# -> nearest neighbor interpolation
ds_grid["data"] = ds_selection.data.variable

ds_grid
```

```{code-cell} python
ds_grid.data.plot(x="x", y="y")
ds.plot.scatter(x="xx", y="yy", c="k")
plt.show()
```

(advanced)=

## Advanced example

This example is based on the Regional Ocean Modeling System (ROMS) [Xarray
example](https://docs.xarray.dev/en/stable/examples/ROMS_ocean_model.html).

```{code-cell} python
ds_roms = xr.tutorial.open_dataset("ROMS_example")
ds_roms
```

The dataset above is represented on a curvilinear grid with 2-dimensional
`lat_rho` and `lon_rho` coordinate variables (in degrees). We will illustrate sampling a
straight line trajectory through this field.

```{code-cell} python
import matplotlib.pyplot as plt

ds_trajectory = xr.Dataset(
    coords={
        "lat": ('trajectory', np.linspace(28, 30, 50)),
        "lon": ('trajectory', np.linspace(-93, -88, 50)),
    },
)

ds_roms.salt.isel(s_rho=-1, ocean_time=0).plot(
    x="lon_rho", y="lat_rho", alpha=0.6
)
plt.plot(
    ds_trajectory.lon.data,
    ds_trajectory.lat.data,
    marker='.',
    color='k',
    ms=4,
    ls="none",
)
plt.show()
```

The default kd-tree structure used by {py:class}`~xarray.indexes.NDPointIndex`
isn't best suited for these latitude and longitude coordinates. Fortunately, there
is a way of using alternative structures. Here let's use {py:class}`sklearn.neighbors.BallTree`,
which supports providing distance metrics such as `haversine` that will better
work with latitude and longitude data.

```{code-cell} python
from sklearn.neighbors import BallTree
from xarray.indexes.nd_point_index import TreeAdapter


class SklearnGeoBallTreeAdapter(TreeAdapter):
    """Works with latitude-longitude values in degrees."""

    def __init__(self, points: np.ndarray, options: dict):
        options.update({'metric': 'haversine'})
        self._balltree = BallTree(np.deg2rad(points), **options)

    def query(self, points: np.ndarray) -> tuple[np.ndarray, np.ndarray]:
        return self._balltree.query(np.deg2rad(points))

    def equals(self, other: "SklearnGeoBallTreeAdapter") -> bool:
        return np.array_equal(self._balltree.data, other._balltree.data)
```

```{note}
Using alternative structures via custom {py:class}`~xarray.indexes.TreeAdapter` subclasses is an
experimental feature!

The adapter above based on {py:class}`sklearn.neighbors.BallTree` will
eventually be available in the [xoak](https://github.com/xarray-contrib/xoak)
package along with other useful adapters.
```

### Assigning

```{code-cell} python
ds_roms_index = ds_roms.set_xindex(
    ("lat_rho", "lon_rho"),
    xr.indexes.NDPointIndex,
    tree_adapter_cls=SklearnGeoBallTreeAdapter,
)
ds_roms_index
```

### Indexing

```{code-cell} python
ds_roms_selection = ds_roms_index.sel(
    lat_rho=ds_trajectory.lat,
    lon_rho=ds_trajectory.lon,
    method="nearest",
)
ds_roms_selection
```

```{code-cell} python
ds_roms.salt.isel(s_rho=-1, ocean_time=0).plot(
    x="lon_rho", y="lat_rho", vmin=0, vmax=35, alpha=0.3
)
plt.scatter(
    ds_trajectory.lon.data,
    ds_trajectory.lat.data,
    c=ds_roms_selection.isel(s_rho=-1, ocean_time=0).salt,
    s=12,
    vmin=0,
    vmax=35,
    edgecolors="k",
    linewidths=0.5,
)
plt.show()
```
