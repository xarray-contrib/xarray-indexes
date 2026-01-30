---
jupytext:
  formats: ipynb,md:myst
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.19.1
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# Nearest neighbors with `NDPointIndex`

```{tip}
**New to spatial trees?** Start with [Why KD-Trees?](why-trees.md) to learn how tree structures enable fast nearest-neighbor search, and when you need alternatives like Ball trees for geographic data.
```

## Highlights

1. {py:class}`xarray.indexes.NDPointIndex` is useful for dealing with
   n-dimensional coordinate variables representing irregular data.
1. It enables efficient point-wise (nearest-neighbors) data selection using Xarray's
   [advanced indexing](https://docs.xarray.dev/en/latest/user-guide/indexing.html#more-advanced-indexing)
   capabilities.
1. By default, a {py:class}`scipy.spatial.KDTree` is used under the hood for
   fast lookup of point data. Although experimental, it is possible to plug in
   alternative structures to `NDPointIndex` (See {ref}`advanced`).

+++

## Basic example: coloring a grid from scattered observations

A common task: you have **scattered observations** and want to **color a regular grid** based on the nearest observation at each grid point. This requires finding the nearest neighbor for every grid cell.

Let's create some scattered measurements:

```{code-cell} ipython3
import numpy as np
import matplotlib.pyplot as plt
import xarray as xr
```

```{code-cell} ipython3
---
tags: [remove-cell]
---
%xmode minimal
xr.set_options(
    display_expand_indexes=True,
    display_expand_data=False,
);
```

```{code-cell} ipython3
shape = (5, 10)
xx = xr.DataArray(
    np.random.uniform(0, 10, size=shape), dims=("y", "x")
)
yy = xr.DataArray(
    np.random.uniform(0, 5, size=shape), dims=("y", "x")
)
data = (xx - 5) ** 2 + (yy - 2.5) ** 2

ds = xr.Dataset(data_vars={"data": data}, coords={"xx": xx, "yy": yy})
ds
```

```{code-cell} ipython3
---
tags: [hide-input]
---
# Show the scattered source points and the empty grid we want to fill
fig, ax = plt.subplots(figsize=(10, 5))

# Draw empty grid lines (the grid we want to color)
for x in range(11):
    ax.axvline(x - 0.5, color='lightgray', lw=0.8, zorder=1)
for y in range(6):
    ax.axhline(y - 0.5, color='lightgray', lw=0.8, zorder=1)

# Plot the scattered source points with colors
scatter = ax.scatter(ds.xx.values.ravel(), ds.yy.values.ravel(),
                     c=ds.data.values.ravel(), s=60, cmap='viridis',
                     edgecolors='black', linewidths=0.5, zorder=5)
plt.colorbar(scatter, ax=ax, label='data')

ax.set_xlim(-0.5, 10.5)
ax.set_ylim(-0.5, 5.5)
ax.set_xlabel('x')
ax.set_ylabel('y')
ax.set_title('Scattered observations + empty grid (goal: color each grid cell)')
plt.tight_layout()
plt.show()
```

### Assigning the index

To enable nearest-neighbor lookups, we attach an `NDPointIndex` to our coordinate variables using `set_xindex`. This builds a KD-tree from the coordinate values, which will be used for efficient spatial queries.

The tuple `("xx", "yy")` tells xarray which coordinates to combine into a multi-dimensional index.

```{note}
The tree is built **once** when you call `set_xindex`. This has a small upfront cost, but all subsequent queries are fast. This makes `NDPointIndex` ideal when you need to query the same dataset many times.
```

```{code-cell} ipython3
ds_index = ds.set_xindex(("xx", "yy"), xr.indexes.NDPointIndex)
ds_index
```

### Querying the index

Now we can query for nearest neighbors using familiar xarray syntax. Under the hood, `.sel(..., method="nearest")` calls `KDTree.query` to efficiently find the closest point:

```{code-cell} ipython3
ds_index.sel(xx=3.4, yy=4.2, method="nearest")
```

### Assigning values from scattered points to a grid

Sometimes you need to transfer scattered observations onto a regular grid. The simplest approach is **nearest-neighbor lookup**: each grid cell gets the value of its closest observation.

```{note}
This is *not* interpolation—there's no averaging or blending. Each grid cell simply takes the value of one source point.
```

Without a spatial index, finding nearest neighbors requires `n_grid × n_points` comparisons (2,500 for 50 grid cells and 50 points). With `NDPointIndex`, each lookup is O(log n).

Let's assign values to a 10×5 grid from our 50 scattered observations:

```{code-cell} ipython3
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

```{code-cell} ipython3
---
tags: [hide-input]
---
# Visualize the result
fig, ax = plt.subplots(figsize=(10, 5))

# Plot the grid
ds_grid.data.plot(x="x", y="y", ax=ax, add_colorbar=True)

# Draw lines connecting each grid cell to its nearest source point
for i, x_val in enumerate(ds_grid.x.values):
    for j, y_val in enumerate(ds_grid.y.values):
        # Find which source point was selected for this grid cell
        nearest_x = ds_selection.xx.sel(x=x_val, y=y_val).values
        nearest_y = ds_selection.yy.sel(x=x_val, y=y_val).values
        # Draw line from grid cell center to source point
        ax.plot([x_val, nearest_x], [y_val, nearest_y],
                color='white', alpha=0.7, lw=1.2, zorder=3)

# Plot the original scattered points on top
ax.scatter(ds.xx.values, ds.yy.values, c='black', s=25, zorder=5,
           edgecolors='white', linewidths=0.5, label='Source points')

ax.set_xlabel('x')
ax.set_ylabel('y')
ax.set_title('Each grid cell takes the value of its nearest source point')
ax.legend(loc='upper right')
plt.tight_layout()
plt.show()
```

(advanced)=

## Advanced example: coloring a trajectory from ocean model data

A real-world use case: you have ocean model output on a curvilinear grid and want to **color a trajectory** based on the nearest model values.

This example uses the Regional Ocean Modeling System (ROMS) [Xarray example](https://docs.xarray.dev/en/stable/examples/ROMS_ocean_model.html). The data is on a grid of `eta_rho` × `xi_rho` (20 × 40 = 800 points), but we want to query by `lat_rho` and `lon_rho`.

```{code-cell} ipython3
ds_roms = xr.tutorial.open_dataset("ROMS_example")
ds_roms
```

### The challenge

We want to color 50 trajectory points based on the nearest of 800 ocean model grid points.

**Without a KD-tree:** 50 × 800 = **40,000 distance calculations**

**With NDPointIndex:** 50 × `~`10 = **~500 calculations** (80x faster!)

```{code-cell} ipython3
---
tags: [hide-input]
---
import matplotlib.pyplot as plt

ds_trajectory = xr.Dataset(
    coords={
        "lat": ("trajectory", np.linspace(28, 30, 50)),
        "lon": ("trajectory", np.linspace(-93, -88, 50)),
    },
)

ds_roms.salt.isel(s_rho=-1, ocean_time=0).plot(
    x="lon_rho", y="lat_rho", alpha=0.6
)
plt.plot(
    ds_trajectory.lon.data,
    ds_trajectory.lat.data,
    marker=".",
    color="k",
    ms=4,
    ls="none",
)
plt.show()
```

### Assigning the Index

First add the index to the lat and lon coord variable using `set_xindex`

```{code-cell} ipython3
ds_roms_index = ds_roms.set_xindex(
    ("lat_rho", "lon_rho"),
    xr.indexes.NDPointIndex,
)
ds_roms_index
```

### Coloring the trajectory

Now we can efficiently find the nearest ocean model point for each trajectory point:

```{code-cell} ipython3
ds_roms_selection = ds_roms_index.sel(
    lat_rho=ds_trajectory.lat,
    lon_rho=ds_trajectory.lon,
    method="nearest",
)
ds_roms_selection
```

The trajectory points are now colored by the salinity of the nearest ocean model grid point - efficiently computed using the KD-tree under the hood:

```{code-cell} ipython3
---
tags: [hide-input]
---
fig, ax = plt.subplots(figsize=(10, 6))

# Plot ocean model salinity as background
ds_roms.salt.isel(s_rho=-1, ocean_time=0).plot(
    x="lon_rho", y="lat_rho", vmin=0, vmax=35, alpha=0.3, ax=ax, add_colorbar=True
)

# Plot trajectory colored by nearest salinity values
scatter = ax.scatter(
    ds_trajectory.lon.data,
    ds_trajectory.lat.data,
    c=ds_roms_selection.isel(s_rho=-1, ocean_time=0).salt,
    s=15,
    vmin=0,
    vmax=35,
    edgecolors="k",
    linewidths=0.5,
)

ax.set_xlabel('Longitude')
ax.set_ylabel('Latitude')
ax.set_title('Ship trajectory colored by nearest ocean model salinity')
plt.tight_layout()
plt.show()
```

## Alternative Tree Data Structures

The default KD-tree uses Euclidean distance, which works well for most cases. However, for **geographic coordinates (lat/lon)**, this can give incorrect results at high latitudes because longitude degrees shrink toward the poles.

```{tip}
See [Why KD-Trees? - The problem with geographic coordinates](why-trees.md#the-problem-with-geographic-coordinates) for a detailed explanation with visualizations.
```

### Using a Ball tree for geographic data

For accurate great-circle distance calculations, you can plug in a Ball tree with the [haversine metric](https://en.wikipedia.org/wiki/Haversine_formula) using a custom {py:class}`~xarray.indexes.TreeAdapter`:

```{code-cell} ipython3
from sklearn.neighbors import BallTree
from xarray.indexes.nd_point_index import TreeAdapter


class SklearnGeoBallTreeAdapter(TreeAdapter):
    """Works with latitude-longitude values in degrees."""

    def __init__(self, points: np.ndarray, options: dict):
        options.update({"metric": "haversine"})
        self._balltree = BallTree(np.deg2rad(points), **options)

    def query(
        self, points: np.ndarray
    ) -> tuple[np.ndarray, np.ndarray]:
        return self._balltree.query(np.deg2rad(points))

    def equals(self, other: "SklearnGeoBallTreeAdapter") -> bool:
        return np.array_equal(
            self._balltree.data, other._balltree.data
        )
```

```{code-cell} ipython3
ds_roms_ball_index = ds_roms.set_xindex(
    ("lat_rho", "lon_rho"),
    xr.indexes.NDPointIndex,
    tree_adapter_cls=SklearnGeoBallTreeAdapter,
)
ds_roms_ball_index
```

### Performance comparison

Let's compare the actual lookup times for our trajectory query:

```{code-cell} ipython3
---
tags: [hide-input]
---
import time

# Get the data points and query points
lat_rho = ds_roms.lat_rho.values.ravel()
lon_rho = ds_roms.lon_rho.values.ravel()
points = np.column_stack([lat_rho, lon_rho])

query_lats = ds_trajectory.lat.values
query_lons = ds_trajectory.lon.values
queries = np.column_stack([query_lats, query_lons])

n_points = len(points)
n_queries = len(queries)

# Time naive search
def naive_nearest(points, queries):
    indices = []
    for q in queries:
        dists = np.sum((points - q)**2, axis=1)
        indices.append(np.argmin(dists))
    return np.array(indices)

start = time.perf_counter()
for _ in range(100):
    naive_nearest(points, queries)
naive_time = (time.perf_counter() - start) / 100 * 1000

# Time KD-tree
start = time.perf_counter()
for _ in range(100):
    ds_roms_index.sel(lat_rho=ds_trajectory.lat, lon_rho=ds_trajectory.lon, method="nearest")
kdtree_time = (time.perf_counter() - start) / 100 * 1000

# Time Ball tree
start = time.perf_counter()
for _ in range(100):
    ds_roms_ball_index.sel(lat_rho=ds_trajectory.lat, lon_rho=ds_trajectory.lon, method="nearest")
balltree_time = (time.perf_counter() - start) / 100 * 1000

# Plot comparison
fig, ax = plt.subplots(figsize=(10, 5))

methods = ['Naive\n(brute force)', 'KD-tree\n(Euclidean)', 'Ball tree\n(haversine)']
times = [naive_time, kdtree_time, balltree_time]
colors = ['gray', 'steelblue', 'coral']

bars = ax.bar(methods, times, color=colors)
ax.set_ylabel('Time per lookup (ms)')
ax.set_title(f'Finding nearest neighbors: {n_queries} trajectory points across {n_points} ocean grid points')

for bar, t in zip(bars, times):
    ax.annotate(f'{t:.2f} ms', (bar.get_x() + bar.get_width()/2, bar.get_height()),
                ha='center', va='bottom', fontsize=11, fontweight='bold')

ax.set_ylim(0, max(times) * 1.2)
plt.tight_layout()
plt.show()

print(f"KD-tree is {naive_time/kdtree_time:.1f}x faster than naive")
print(f"Ball tree is {naive_time/balltree_time:.1f}x faster than naive")
print("\nThe tradeoff: KD-tree is faster but treats lat/lon as flat.")
print("Ball tree is slower but uses haversine for correct great-circle distances.")
```

## Summary: choosing the right tree

| Use case                               | Recommended              | Why                                           |
| -------------------------------------- | ------------------------ | --------------------------------------------- |
| Cartesian coordinates (x, y in meters) | KD-tree (default)        | Fastest option, Euclidean distance is correct |
| Geographic data near the equator       | KD-tree                  | Euclidean on lat/lon is approximately correct |
| Geographic data at high latitudes      | Ball tree with haversine | Euclidean on lat/lon gives wrong answers      |
| Custom distance metric needed          | Ball tree                | Supports haversine, Manhattan, and others     |
| Very high dimensions (>20)             | Ball tree                | KD-trees degrade in high dimensions           |

```{seealso}
[Why KD-Trees?](why-trees.md) for a visual explanation of how these tree structures work and why haversine matters for geographic data.
```
