---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# Functional transformations with `CoordinateTransformIndex`

## Highlights

1. The coordinate variables whose values are described by _coordinate
   transformations_ -- i.e., by a set of formulas describing the relationship
   between array indices and coordinate labels -- are best handled via an
   {py:class}`xarray.indexes.CoordinateTransformIndex`.
1. Coordinate variables associated with such index are lazy and therefore
   use very little memory. They may have arbitrary dimensions.
1. Alignment may be implemented in an optimal way, i.e., based on coordinate transformation
   parameters rather than on raw coordinate labels.
1. Xarray exposes an abstract {py:class}`~xarray.indexes.CoordinateTransform`
   class to plug in 3rd-party coordinate transformations with support
   of dimension and coordinate variable names (see the example below).

```{seealso}
`CoordinateTransformIndex` is often used as a building block by other
custom indexes such as {py:class}`xarray.indexes.RangeIndex` (see
{doc}`../builtin/range`) and {py:class}`rasterix.RasterIndex` (see
{doc}`../earth/raster`).
```

## Example (Astronomy)

As a real-world example, let's create a custom
{py:class}`xarray.indexes.CoordinateTransform` that wraps an
{py:class}`astropy.wcs.WCS` object. This Xarray coordinate transform adapter
class simply maps Xarray dimension and coordinate variable names to pixel
and world axis names of the [shared Python interface for World Coordinate
Systems](https://doi.org/10.5281/zenodo.1188874) used in Astropy.

```{note}
This example is taken and adapted from
[this gist](https://gist.github.com/Cadair/4a03750868e044ac4bdd6f3a04ed7abc) by
[Stuart Mumford](https://github.com/Cadair).

It only provides basic integration between Astropy's WCS and Xarray
coordinate transforms. More advanced integration could leverage the
[slicing capabilities of WCS objects](https://docs.astropy.org/en/latest/wcs/wcsapi.html#slicing-of-wcs-objects)
in a custom {py:class}`xarray.indexes.CoordinateTransformIndex` subclass.

```

```{code-cell} python
from collections.abc import Hashable
from typing import Any

import numpy as np
import xarray as xr
from astropy.wcs import WCS


def escape(name):
    return name.replace(".", "_").replace("custom:", "")


class WCSCoordinateTransform(xr.indexes.CoordinateTransform):
    """Lightweight adapter class for the World Coordinate Systems (WCS) API.

    More info: https://docs.astropy.org/en/latest/wcs/wcsapi.html
    """

    def __init__(self, wcs: WCS):
        pixel_axis_names = [
            pan or f"dim{i}"
            for i, pan in enumerate(wcs.pixel_axis_names)
        ]
        world_axis_names = [
            escape(wan or wphy or f"coord{i}")
            for i, (wan, wphy) in enumerate(
                zip(
                    wcs.world_axis_names,
                    wcs.world_axis_physical_types,
                )
            )
        ]
        dim_size = {
            name: size
            for name, size in zip(pixel_axis_names, wcs.array_shape)
        }

        super().__init__(
            world_axis_names, dim_size, dtype=np.dtype(float)
        )
        self.wcs = wcs

    def forward(
        self, dim_positions: dict[str, Any]
    ) -> dict[Hashable, Any]:
        """Perform array -> world coordinate transformation."""

        pixel = [dim_positions[dim] for dim in self.dims]
        world = self.wcs.array_index_to_world_values(*pixel)

        return {name: w for name, w in zip(self.coord_names, world)}

    def reverse(
        self, coord_labels: dict[Hashable, Any]
    ) -> dict[str, Any]:
        """Perform world -> array coordinate reverse transformation."""

        world = [coord_labels[name] for name in self.coord_names]
        pixel = self.wcs.world_to_array_index_values(*world)

        return {name: p for name, p in zip(self.dims, pixel)}
```

```{code-cell} python
---
tags: [remove-cell]
---
%xmode minimal
xr.set_options(
    display_expand_indexes=True,
    display_expand_data=True,
)
np.set_printoptions(precision=3, threshold=10, edgeitems=2)
```

### Assigning

Let's now create a small function that opens a FITS file with Astropy, creates
an Xarray {py:class}`~xarray.indexes.CoordinateTransformIndex` and its
associated lazy coordinate variables from the {py:class}`~astropy.wcs.WCS`
object and returns both the data and coordinates as an
{py:class}`xarray.DataArray`.

```{code-cell} python
from astropy.io import fits
from astropy.utils.data import get_pkg_data_filename


def open_fits_dataarray(filename, item=0):
    hdu = fits.open(filename)[item]
    wcs = WCS(hdu.header)

    transform = WCSCoordinateTransform(wcs)
    index = xr.indexes.CoordinateTransformIndex(transform)
    coords = xr.Coordinates.from_xindex(index)

    return xr.DataArray(
        hdu.data,
        coords=coords,
        dims=transform.dims,
        attrs={"wcs": wcs},
    )
```

Open a simple image with two celestial axes.

```{code-cell} python
fname = get_pkg_data_filename("galactic_center/gc_2mass_k.fits")

da_2d = open_fits_dataarray(fname)
da_2d
```

```{code-cell} python
# lazy coordinate variables!

da_2d.pos_eq_ra
```

```{code-cell} python
da_2d.plot.pcolormesh(
    x="pos_eq_ra",
    y="pos_eq_dec",
    vmax=1300,
    cmap="magma",
);
```

Open a spectral cube with two celestial axes and one spectral axis.

```{code-cell} python
fname = get_pkg_data_filename("l1448/l1448_13co.fits")

da_3d = open_fits_dataarray(fname)
da_3d
```
