---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# Units with `pint_xarray.PintIndex`

````{grid}
```{grid-item}
:columns: 3
```{image} https://pint.readthedocs.io/en/latest/_static/logo-full.jpg
---
alt: pint logo
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

First we open the dataset, fill in missing `units` attributes, and calculate the length of the vectors for later:

```{code-cell} python
%xmode minimal
import numpy as np
import xarray as xr

xr.set_options(
    display_expand_indexes=True, display_expand_attrs=False, display_expand_data=False
)

ds = (
    xr.tutorial.open_dataset("eraint_uvz")
    .load()
    .assign_coords(month=lambda ds: ds["month"].assign_attrs({"units": "months"}))
    .assign(windspeed=lambda ds: np.hypot(ds["u"], ds["v"]))
)
ds
```

### Quantifying

Now we can quantify to convert arrays with a `"units"` attribute to quantity arrays:

```{code-cell} python
import cf_xarray.units
import pint_xarray

quantified = ds.pint.quantify()
quantified
```

Note how all variables are associated with a {py:class}`pint.Quantity` array, and how all coordinate variables are associated with a {py:class}`pint_xarray.PintIndex` wrapping a `PandasIndex`.

### Selection

With the `PintIndex`, selecting with quantities will convert the indexers to the index' units:

```{code-cell} python
ureg = pint_xarray.unit_registry

quantified.sel(
    latitude=slice(ureg.Quantity(4800, "arcmin"), ureg.Quantity(600, "arcmin")),
    longitude=slice(ureg.Quantity(-10, "degree"), ureg.Quantity(np.pi, "radians")),
)
```

or raise on incompatible units:

```{code-cell} python
---
tags: [raises-exception]
---
quantified.sel(month=ureg.Quantity(10, "m"))
```

### Numerical operations

We can also perform numerical operations, like integration:

```{code-cell} python
quantified["windspeed"].integrate("month")
```

Note how the units are displayed as `"meter * months / second"` and not the expected `"meter"`? This is caused by `pint` trying avoid implicit conversions as much as possible, which can substantially reduce the amount of computations.
