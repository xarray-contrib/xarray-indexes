---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# rasterix: RasterIndex

````{grid}
```{grid-item}
:columns: 3
```{image} https://rasterix.readthedocs.io/en/latest/_static/rasterix.png
---
alt: Alt text
width: 200px
align: center
---
```
```{grid-item}
:columns: 9
```{seealso}
Learn more at the [Rasterix](https://rasterix.readthedocs.io/en/latest/) documentation page.
```
````

## Highlights

Rasterix provides a RasterIndex that allows indexing using a functional transformation defined by an _affine transform_.

It uses {py:class}`~xarray.indexes.CoordinateTransformIndex` as a building block. In doing so,

1. RasterIndex eliminates an entire class of bugs where Xarray allows you to add (for example) two datasets with different affine transforms (and/or projections) and return nonsensical outputs.
1. The associated coordinate variables are lazy, and use very little memory. Thus very large coordinate frames can be represented.

Rasterix uses a function {py:func}`rasterix.assign_index`.

## Example

Here is a GeoTIFF file opened with rioxarray. GeoTIFF files _do not contain explicit coordinate arrays_, instead they commonly store the coefficients of an affine transform that software libraries use to calculate the coordinates.

On reading, we set `"parse_coordinates": False` to tell rioxarray to not generate coordinate variables.

```{code-cell}
import rasterix
import xarray as xr

xr.set_options(display_expand_indexes=True, display_expand_attrs=False, display_expand_data=False)

source = "https://noaadata.apps.nsidc.org/NOAA/G02135/south/daily/geotiff/2024/01_Jan/S_20240101_concentration_v3.0.tif"

da = xr.open_dataarray(source, engine="rasterio", backend_kwargs={"parse_coordinates": False})
da
```

Notice how there are two dimensions `x` and `y`, but no coordinates associated with them.,

The affine transform information is stored in the attributes of `spatial_ref` under the name `"GeoTransform"`

```{code-cell}
da.spatial_ref.attrs
```

### Assignment

Rasterix provides a helpful `assign_index` function to automate the process of creating an index

```{code-cell}
da = rasterix.assign_index(da)
da
```

We now have coordinate values, lazily generated on demand!

### Indexing

Slicing this dataset preserves the RasterIndex, though with a new transform

```{code-cell}
sliced = da.isel(x=slice(100, 200), y=slice(200, 300))
sliced
```

Compare the underlying transforms:

```{code-cell}
sliced.xindexes["x"].transform(), da.xindexes["x"].transform()
```

### Combining

The affine transform is also used for alignment, and combining!

Here is a simple example. We slice the input dataset in to two

```{code-cell}
left = da.isel(x=slice(150))
right = da.isel(x=slice(150, None))
```

Let's look at the bounding boxes for the sliced datasets

```{code-cell}
---
tags: [hide-input]
---
import matplotlib.pyplot as plt

def plot_bbox(bbox):
    x0, y0, x1, y1 = bbox
    plt.plot([x0, x0, x1, x1, x0], [y0, y1, y1, y0, y0])

plot_bbox(left.xindexes["x"].bbox)
plot_bbox(right.xindexes["x"].bbox)
```

Concatenating these two along x preserves the RasterIndex!

```{code-cell}
combined = xr.concat([left, right], dim="x")
combined
```

```{code-cell}
---
tags: [hide-input]
---
plot_bbox(combined.xindexes["x"].bbox)
```

The coordinates on the combined dataset is equal to the original dataset

```{code-cell}
combined.xindexes["x"].equals(da.xindexes["x"])
```

This functionality extends to [multi-dimsensional tiling](https://rasterix.readthedocs.io/en/latest/raster_index/combining.html#combine-nested) using {py:func}`xarray.combine_nested` too!

### Tracking the affine transform

Let's reopen the same GeoTIFF file using rioxarray (without assigning a raster
index), slice both the `x` and `y` dimensions with steps > 1 and get the affine
transform parameters via rioxarray:

```{code-cell}
temp = xr.open_dataarray(source, engine="rasterio")
subset_a = temp.isel(x=slice(100, 200, 10), y=slice(200, 300, 20))
subset_a.rio.transform()
```

Rioxarray calculates the affine parameters of a dataarray by looking at the
coordinate data and/or metadata. It maybe caches the results to avoid expensive
computation, although in some cases like here a re-calculation seems to be
needed:

```{code-cell}
subset_a.rio.transform(recalc=True)
```

The raster index added above eliminates the need to (re)calculate and/or cache
the affine parameters from coordinate data after an operation on the dataaarray.
Instead, it tracks and compute affine parameter values during the operation and
generates (on-demand) coordinate data from these new parameter values:

```{code-cell}
subset_b = da.isel(x=slice(100, 200, 10), y=slice(200, 300, 20))

subset_b.xindexes["x"].transform()
```
