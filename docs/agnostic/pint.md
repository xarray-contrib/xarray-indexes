---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# PintIndex

````{grid}
```{grid-item}
:columns: 3
```{image} https://pint.readthedocs.io/en/latest/_static/logo-full.png
---
alt: pint-xarray logo
width: 200px
align: center
---
```
```{grid-item}
:columns: 9
```{seealso}
Learn more at the [pint-xarray](https://pint-xarray.readthedocs.io/en/latest/) documentation page.
```
````

## Highlights

`pint-xarray` provides an index that wraps other indexes and attaches units to the indexed coordinates. This allows operations like `integrate` or `sel` to take the units into account.

## Example

First we open the dataset and perform some preprocessing:

```{python}
import numpy as np
import xarray as xr

xr.set_options(
    display_expand_indexes=True, display_expand_attrs=False, display_expand_data=False
)

ds = (
    xr.tutorial.open_dataset("eraint_uvz")
    .load()
    .assign_coords(months=lambda ds: ds["months"].assign_attrs({"units": "months"}))
    .assign(windspeed=lambda ds: np.hypot(ds["u"], ds["v"]))
)
ds
```

### Quantifying

Now we can quantify to convert arrays with a `"units"` attribute to quantity arrays:

```{python}
import cf_xarray.units
import pint_xarray

quantified = ds.pint.quantify()
quantified
```

Note how all variables are associated with a {py:class}`pint.Quantity` array, and how all coordinate variables are associated with a {py:class}`pint_xarray.PintIndex` wrapping a `PandasIndex`.

### Selection

With the `PintIndex`, selecting with quantities will convert the indexers to the index' units:

```{python}
quantified.sel(
    latitude=slice(ureg.Quantity(4800, "arcmin"), ureg.Quantity(600, "arcmin")),
    longitude=slice(ureg.Quantity(-10, "degree"), ureg.Quantity(np.pi, "radians")),
)
```

or raise on incompatible units:

```{python}
quantified.sel(
    months=ureg.Quantity(10, "m"),
    level=200,
)
```

(`pint` considers values without units as "dimensionless")

### Numerical operations

We can also perform numerical operations, like integration:

```{python}
quantified["windspeed"].integrate("months")
```

Note how the units are displayed as `"meter * months / second"` and not the expected `"meter"`? This is caused by `pint` trying avoid implicit conversions as much as possible, which can substantially reduce the amount of computations.
