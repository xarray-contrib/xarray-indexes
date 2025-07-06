---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# xvec: GeometryIndex

````{grid} 12
```{grid-item}
:columns: 4
```{image} https://xvec.readthedocs.io/en/stable/_images/logo.svg
---
alt: Alt text
width: 600px
align: center
---
```
```{grid-item}
:columns: 8
```{seealso}
Learn more at the [xvec](https://xvec.readthedocs.io) documentation.
```
````

## Highlights

Xvec's use of custom indexes is exciting because it illustrates how a new Index can help define a new data model --- _vector data cube_: "an n-D array that has either at least one dimension indexed by a 2-D array of vector geometries".

1. Indexing using geometries and associated predicates is supported using `.sel`
1. A new `.xvec` accessor exposes additional querying functionality.
1. Exposes complex functionality from other full-featured packages (e.g. shapely) to Xarray.

## Example

First we create a data cube with geometries

```{code-cell}
---
tags: [hide-input]
---
import geopandas as gpd
from geodatasets import get_path
import shapely
import xarray as xr
import xvec

xr.set_options(display_expand_indexes=True, display_expand_data=False, display_expand_attrs=False)

counties = gpd.read_file(get_path("geoda.natregimes"))

cube = xr.Dataset(
    data_vars=dict(
        population=(["county", "year"], counties[["PO60", "PO70", "PO80", "PO90"]]),
        unemployment=(["county", "year"], counties[["UE60", "UE70", "UE80", "UE90"]]),
        divorce=(["county", "year"], counties[["DV60", "DV70", "DV80", "DV90"]]),
        age=(["county", "year"], counties[["MA60", "MA70", "MA80", "MA90"]]),
    ),
    coords=dict(county=counties.geometry, year=[1960, 1970, 1980, 1990]),
)
cube
```

Note how the `county` dimension is associated with a {py:class}`geopandas.GeometryArray`.

### Assigning

Now we can assign a {py:class}`xvec.GeometryIndex` to `county`.

```{code-cell}
cube = cube.xvec.set_geom_indexes("county")
cube
```

### Indexing

#### Geometries as labels

```{code-cell}
cube.sel(county=cube.county[0])
```

#### Complex spatial predicates

Lets index to counties that intersect the provided bounding box

```{code-cell}
box = shapely.box(-97, 45, -99, 48)

subset = cube.sel(county=box, method="intersects")
subset
```

Notice how we did that with {py:meth}`xarray.DataArray.sel`?!

```{code-cell}
f, axes = subset.population.xvec.plot(col="year")
for ax in axes.flat:
    ax.plot(*box.boundary.xy, color='w')
```
