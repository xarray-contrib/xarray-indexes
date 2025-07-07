---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# pandas: IntervalIndex

````{grid}
```{grid-item}
:columns: 3
```{image} https://pandas.pydata.org/docs/_static/pandas.svg
---
alt: Alt text
width: 200px
align: center
---
```
```{grid-item}
:columns: 9
```{seealso}
Learn more at the [Pandas](https://pandas.pydata.org/pandas-docs/stable/user_guide/advanced.html#intervalindex) documentation.
```
````

# Highlights

1. Xarray's built-in support for pandas Index classes extends to more sophisticated classes like {py:class}`pandas.IntervalIndex`.
1. Xarray now generates such indexes automatically when using {py:meth}`xarray.DataArray.groupby_bins` or {py:meth}`xarray.Dataset.groupby_bins`.
1. Sadly {py:class}`pandas.IntervalIndex` supports numpy datetimes but not [cftime](https://unidata.github.io/cftime/).

```{important}
A pandas IntervalIndex models intervals using a single variable. The [Climate and Forecast Conventions](https://cfconventions.org/Data/cf-conventions/cf-conventions-1.11/cf-conventions.html#cell-boundaries), by contrast, model the intervals using  two arrays: the intervals ("bounds" variable) and "central values".
```

## Example

### Assigning

```{code-cell}
%xmode minimal

import pandas as pd
import xarray as xr

xr.set_options(display_expand_indexes=True, display_expand_attrs=False)
pd.set_option('display.max_seq_items', 10)

orig = xr.tutorial.open_dataset("air_temperature")
orig
```

Let's replace the `time` vector with an IntervalIndex, assuming that the data represent averages over 6 hour periods centered at 00h, 06h, 12h, 18h

```{code-cell}
left = orig.time.data - pd.Timedelta("3h")
right = orig.time.data + pd.Timedelta("3h")
time_bounds = pd.IntervalIndex.from_arrays(left, right, closed="left")
time_bounds
```

```{code-cell}
indexed = orig.copy(deep=True)
indexed["time"] = time_bounds
indexed
```

Note the above object still shows the `time` coordinates has associated `PandasIndex` but the values are now represented in and "IntervalArray" (as indicated by `interval[datetime64[ns], left]`)

### Indexing

Let's index out a representative value for 2013-05-01 02:00.

```{code-cell}
---
tags: [raises-exception]
---
orig.sel(time="2013-05-01 02:00")
```

Indexing the original dataset required specifying `method="nearest"`

```{code-cell}
orig.sel(time="2013-05-01 02:00", method="nearest").time
```

With an IntervalIndex, however, that is unnecessary

```{code-cell}
indexed.sel(time="2013-05-01 02:00").time
```

### Binned grouping

Xarray now creates IntervalIndex by default for binned grouping operations

```{code-cell}
orig.groupby_bins("lat", bins=[25, 35, 45, 55]).mean()
```
