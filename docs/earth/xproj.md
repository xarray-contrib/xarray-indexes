---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# Coordinate Reference Systems with `xproj.CRSIndex`

```{seealso}
Learn more at the [xproj](https://xproj.readthedocs.io) documentation
```

## Highlights

1. `xproj` provides a `CRSIndex` to handle "coordinate reference system" (CRS) information.
1. CRS information is commonly stored in the attributes of a _scalar_ coordinate variable, usually called `"spatial_ref"` or one that is annotated as a "grid mapping variable" according to the CF conventions.
1. The purpose of `CRSIndex` is to handle _alignment_, not indexing. That is, we would like errors to be raised when adding or concatenating (for example) two datasets with different CRS. Such operations are nonsensical without reprojecting the data to a common CRS.

## Example

### Assigning

```{code-cell}
%xmode minimal

import xarray as xr
import xproj  # registers the .proj accessor

xr.set_options(display_expand_data=False, display_expand_attrs=False, display_expand_indexes=True)

ds = xr.tutorial.load_dataset("air_temperature")
ds
```

This dataset has no CRS information.
`xproj` uses Xarray's accessor interface to provide functionality. We use that to assign a CRS --- the common WGS84 global geodetic CRS ([EPSG code: 4326](https://epsg.io/4326)).

```{code-cell}
ds_wgs84 = ds.proj.assign_crs(spatial_ref="epsg:4326")
ds_wgs84
```

Note the CRS information in the repr for CRSIndex associated with `spatial_ref` above.

### Alignment

_Alignment_ means making sure the objects conform to the same coordinate reference frame.
When executing multi-object operations (e.g. binary operations, concatenation, merging etc.), Xarray automatically aligns these objects using Indexes.

To illustrate we will artificially create a new dataset with a different CRS:

```{code-cell}
ds_wgs72 = ds_wgs84.proj.assign_crs(spatial_ref="epsg:4322", allow_override=True)
ds_wgs72
```

Now lets add the two:

```{code-cell}
---
tags: [raises-exception]
---
ds_wgs84 + ds_wgs72
```

How about concatenating?

```{code-cell}
---
tags: [raises-exception]
---
xr.concat([ds_wgs84, ds_wgs72], dim="newdim")
```
