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

## Example

Taken and adapted from
[here](https://gist.github.com/Cadair/4a03750868e044ac4bdd6f3a04ed7abc) (author:
Stuart Mumford).

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
    def __init__(
        self,
        wcs: WCS,
    ):
        pixel_axis_names = [
            pan or f"dim{i}"
            for i, pan in enumerate(wcs.pixel_axis_names)
        ]
        world_axis_names = [
            escape(wan or wphy or f"coord{i}")
            for i, (wan, wphy) in enumerate(
                zip(wcs.world_axis_names, wcs.world_axis_physical_types)
            )
        ]
        dim_size = {
            name: size
            for name, size in zip(pixel_axis_names, wcs.array_shape)
        }

        super().__init__(world_axis_names, dim_size, dtype=np.dtype(float))
        self.wcs = wcs

    def forward(self, dim_positions: dict[str, Any]) -> dict[Hashable, Any]:
        """Perform array -> world coordinate transformation."""

        pixel = [dim_positions[dim] for dim in self.dims]
        world = self.wcs.array_index_to_world_values(*pixel)

        return {name: w for name, w in zip(self.coord_names, world)}

    def reverse(self, coord_labels: dict[Hashable, Any]) -> dict[str, Any]:
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
    display_expand_attrs=False,
);
```

```{code-cell} python
from astropy.io import fits
from astropy.utils.data import get_pkg_data_filename


filename = get_pkg_data_filename('l1448/l1448_13co.fits')
hdu = fits.open(filename)[0]
wcs = WCS(hdu.header)
data = hdu.data

transform = WCSCoordinateTransform(wcs)
index = xr.indexes.CoordinateTransformIndex(transform)
coords = xr.Coordinates.from_xindex(index)

ds = xr.Dataset({'primary': (transform.dims, data)}, coords=coords)
ds
```

```{code-cell} python
da_subset = ds.primary.isel(dim2=[45, 50, 55])

da_subset.plot.pcolormesh(
    x="pos_eq_ra",
    y="spect_dopplerVeloc_opt",
    col="dim2",
    col_wrap=2,
);
```

```{code-cell} python
sliced = ds.isel(dim2=50)

ds.primary.isel(dim2=50).plot(subplot_kws=dict(projection=wcs, slices=(50, 'x', 'y')))
```

```{code-cell} python
filename2 = get_pkg_data_filename('galactic_center/gc_2mass_k.fits')
hdu2 = fits.open(filename2)[0]
wcs2 = WCS(hdu2.header)
data2 = hdu2.data

transform2 = WCSCoordinateTransform(wcs2)
index2 = xr.indexes.CoordinateTransformIndex(transform2)
coords2 = xr.Coordinates.from_xindex(index2)

ds2 = xr.Dataset({'primary': (transform2.dims, data2)}, coords=coords2)
ds2
```

```{code-cell} python
ds2.primary.plot.pcolormesh(x="pos_eq_ra", y="pos_eq_dec");
```

```{code-cell} python
hdu2 = fits.open("http://data.sunpy.org/sunpy/v1/AIA20110607_063302_0171_lowres.fits")[1]
wcs2 = WCS(hdu2.header)
data2 = hdu2.data

transform2 = WCSCoordinateTransform(wcs2)
index2 = xr.indexes.CoordinateTransformIndex(transform2)
coords2 = xr.Coordinates.from_xindex(index2)

ds2 = xr.Dataset({'data': (transform2.dims, data2)}, coords=coords2)
ds2
```

```{code-cell} python
ds2.data.plot.pcolormesh(x="pos_helioprojective_lon", y="pos_helioprojective_lat");
```

```{code-cell} python
ds2.data.plot(subplot_kws=dict(projection=wcs2), vmin=0, vmax=2000)
```
