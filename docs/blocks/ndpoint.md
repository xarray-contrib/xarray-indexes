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
**New to spatial trees?** Start with [Tree-Based Indexing](https://tutorial.xarray.dev/advanced/indexing/why-trees.html) to learn how tree structures enable fast nearest-neighbor search, and when you need alternatives like Ball trees for geographic data.
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

## Basic usage

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

```{code-cell} ipython3
---
tags: [hide-input]
---
query_x, query_y = 3.4, 4.2
result = ds_index.sel(xx=query_x, yy=query_y, method="nearest")
nearest_x = result.xx.item()
nearest_y = result.yy.item()

fig, ax = plt.subplots(figsize=(10, 5))

# All points at low alpha
ax.scatter(
    ds.xx.values.ravel(), ds.yy.values.ravel(),
    c='steelblue', s=40, alpha=0.2, edgecolors='gray', linewidths=0.5, zorder=3,
)

# Line connecting query to nearest
ax.annotate(
    '', xy=(nearest_x, nearest_y), xytext=(query_x, query_y),
    arrowprops=dict(arrowstyle='-', color='black', lw=1.5, ls='--'),
    zorder=4,
)

# Nearest point highlighted
ax.scatter(
    nearest_x, nearest_y,
    c='steelblue', s=120, edgecolors='black', linewidths=1.5, zorder=6,
    label=f'nearest ({nearest_x:.1f}, {nearest_y:.1f})',
)

# Query point
ax.scatter(
    query_x, query_y,
    c='red', s=120, marker='X', linewidths=0.5, edgecolors='darkred', zorder=6,
    label=f'query ({query_x}, {query_y})',
)

ax.set_xlim(-0.5, 10.5)
ax.set_ylim(-0.5, 5.5)
ax.set_xlabel('x')
ax.set_ylabel('y')
ax.set_title('Nearest-neighbor lookup')
ax.legend()
plt.tight_layout()
plt.show()
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

## Advanced usage

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

## Alternative trees

The default KD-tree uses Euclidean distance, which works well for most cases. However, for **geographic coordinates (lat/lon)**, this can give incorrect results at high latitudes because longitude degrees shrink toward the poles.

```{tip}
See [Tree-Based Indexing - The problem with geographic coordinates](https://tutorial.xarray.dev/advanced/indexing/why-trees.html#the-problem-with-geographic-coordinates) for a detailed explanation with visualizations.
```

### Using a Ball tree for geographic data

For accurate great-circle distance calculations, you can plug in a Ball tree with the [haversine metric](https://en.wikipedia.org/wiki/Haversine_formula). The [xoak](https://xoak.readthedocs.io) package provides ready-made tree adapters for common use cases, including {class}`~xoak.SklearnGeoBallTreeAdapter` which wraps scikit-learn's `BallTree` with the haversine metric.

Pass it as the `tree_adapter_cls` argument to `set_xindex` — everything else stays the same:

```{code-cell} ipython3
from xoak import SklearnGeoBallTreeAdapter

ds_roms_ball_index = ds_roms.set_xindex(
    ("lat_rho", "lon_rho"),
    xr.indexes.NDPointIndex,
    tree_adapter_cls=SklearnGeoBallTreeAdapter,
)
ds_roms_ball_index
```

The adapter handles converting lat/lon from degrees to radians internally, so you can query with the same degree-valued coordinates as before.

At high latitudes, longitude degrees are much shorter than latitude degrees. A KD-tree and Ball tree can pick **different** nearest neighbors for the same query:

```{code-cell} ipython3
---
tags: [hide-input]
---
# Two points at 80°N: one east (along longitude), one north (along latitude)
# The query sits between them — closer in *degrees* to the north point,
# but closer in *km* to the east point.
point_a = (80.0, 2.0)   # 2° east of query
point_b = (80.5, 0.0)   # 0.5° north of query
query = (80.0, 0.0)

# Use 2D coords on a dummy grid (NDPointIndex requires shared dims)
ds_demo = xr.Dataset(
    {"label": (("i", "j"), [["A", "B"]])},
    coords={
        "lat": (("i", "j"), [[point_a[0], point_b[0]]]),
        "lon": (("i", "j"), [[point_a[1], point_b[1]]]),
    },
)

# KD-tree (Euclidean on degrees)
ds_kd = ds_demo.set_xindex(("lat", "lon"), xr.indexes.NDPointIndex)
result_kd = ds_kd.sel(lat=query[0], lon=query[1], method="nearest")

# Ball tree (haversine)
ds_ball = ds_demo.set_xindex(
    ("lat", "lon"), xr.indexes.NDPointIndex,
    tree_adapter_cls=SklearnGeoBallTreeAdapter,
)
result_ball = ds_ball.sel(lat=query[0], lon=query[1], method="nearest")

kd_pick = (result_kd.lat.item(), result_kd.lon.item())
ball_pick = (result_ball.lat.item(), result_ball.lon.item())

km_per_deg_lon = 111.0 * np.cos(np.radians(80.0))
km_to_a = abs(query[1] - point_a[1]) * km_per_deg_lon
km_to_b = abs(query[0] - point_b[0]) * 111.0

# Convert to km relative to query
def to_km(lat, lon):
    return (lon - query[1]) * km_per_deg_lon, (lat - query[0]) * 111.0

a_km = to_km(*point_a)
b_km = to_km(*point_b)
q_km = (0.0, 0.0)

fig, (ax_deg, ax_km) = plt.subplots(1, 2, figsize=(13, 5))

for ax, coords, xlabel, ylabel, title, a_label, b_label in [
    (ax_deg,
     {"a": (point_a[1], point_a[0]), "b": (point_b[1], point_b[0]), "q": (query[1], query[0])},
     "Longitude (°)", "Latitude (°)",
     "In degrees: B looks closer",
     f"A (2.0°)", f"B (0.5°)"),
    (ax_km,
     {"a": (a_km[0], a_km[1]), "b": (b_km[0], b_km[1]), "q": q_km},
     "East-West (km)", "North-South (km)",
     "In km: A is actually closer",
     f"A ({km_to_a:.0f} km)", f"B ({km_to_b:.0f} km)"),
]:
    # Line to A (Ball tree / haversine picks this)
    ax.annotate(
        "", xy=coords["a"], xytext=coords["q"],
        arrowprops=dict(arrowstyle="-", color="coral", lw=2, ls="--"), zorder=4,
    )
    # Line to B (KD-tree picks this)
    ax.annotate(
        "", xy=coords["b"], xytext=coords["q"],
        arrowprops=dict(arrowstyle="-", color="steelblue", lw=2, ls="--"), zorder=4,
    )

    # Label the lines
    mid_a = ((coords["q"][0] + coords["a"][0]) / 2, (coords["q"][1] + coords["a"][1]) / 2)
    mid_b = ((coords["q"][0] + coords["b"][0]) / 2, (coords["q"][1] + coords["b"][1]) / 2)
    ax.annotate("haversine", mid_a, xytext=(8, -10), textcoords="offset points",
                fontsize=9, color="coral", fontstyle="italic")
    ax.annotate("KD-tree", mid_b, xytext=(8, 4), textcoords="offset points",
                fontsize=9, color="steelblue", fontstyle="italic")

    # Points
    ax.scatter(*coords["a"], c="coral", s=140, edgecolors="black", linewidths=1.5, zorder=6)
    ax.scatter(*coords["b"], c="steelblue", s=140, edgecolors="black", linewidths=1.5, zorder=6)
    ax.scatter(*coords["q"], c="red", s=180, marker="X",
               edgecolors="darkred", linewidths=0.5, zorder=7)

    # Point labels
    ax.annotate(a_label, coords["a"], xytext=(12, -14), textcoords="offset points",
                fontsize=10, fontweight="bold")
    ax.annotate(b_label, coords["b"], xytext=(12, 8), textcoords="offset points",
                fontsize=10, fontweight="bold")
    ax.annotate("query", coords["q"], xytext=(12, -14), textcoords="offset points",
                fontsize=10, color="red", fontweight="bold")

    ax.set_xlabel(xlabel)
    ax.set_ylabel(ylabel)
    ax.set_title(title, fontweight="bold")
    ax.grid(True, alpha=0.3)

# Match x and y limits so distances are visually comparable
deg_lim = max(abs(point_a[1] - query[1]), abs(point_b[0] - query[0])) * 1.3
ax_deg.set_xlim(-deg_lim * 0.15, deg_lim)
ax_deg.set_ylim(query[0] - deg_lim * 0.15, query[0] + deg_lim)

km_lim = max(abs(a_km[0]), abs(a_km[1]), abs(b_km[0]), abs(b_km[1])) * 1.3
ax_km.set_xlim(-km_lim * 0.15, km_lim)
ax_km.set_ylim(-km_lim * 0.15, km_lim)

plt.tight_layout()
plt.show()
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

## Choosing a tree

| Use case                               | Recommended              | Why                                           |
| -------------------------------------- | ------------------------ | --------------------------------------------- |
| Cartesian coordinates (x, y in meters) | KD-tree (default)        | Fastest option, Euclidean distance is correct |
| Geographic data near the equator       | KD-tree                  | Euclidean on lat/lon is approximately correct |
| Geographic data at high latitudes      | Ball tree with haversine | Euclidean on lat/lon gives wrong answers      |
| Custom distance metric needed          | Ball tree                | Supports haversine, Manhattan, and others     |
| Very high dimensions (>20)             | Ball tree                | KD-trees degrade in high dimensions           |

```{seealso}
- [Tree-Based Indexing](https://tutorial.xarray.dev/advanced/indexing/why-trees.html) for a visual explanation of how these tree structures work and why haversine matters for geographic data.
- [Building custom tree adapters](https://xoak.readthedocs.io/en/latest/examples/custom_indexes.html) for advanced use cases like implementing your own {py:class}`~xarray.indexes.TreeAdapter`.
```
