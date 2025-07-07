# Xarray Custom Indexes Gallery

## Background

Xarray's data model was initially heavily inspired from the
[NetCDF](https://www.unidata.ucar.edu/software/netcdf/) file format, making it
well suited for working with n-dimensional, rectilinear gridded datasets
commonly found in scientific data analysis, especially in the geosciences.

```{figure} _static/figs/xarray-dataset-diagram-legacy.png
:alt: Xarray data model
:align: center
:width: 500px
:class: dark-light
An illustration of the Xarray data model.
```

Over the years, Xarray has been used in an increasing number of domains as a
convenient, general-purpose Python library for handling n-dimensional labeled
arrays. Xarray's data structures are now reused for representing a wide range of
datasets including sparse data, curvilinear or irregular grids, staggered grids,
discrete global grids and vector data cubes.

```{figure} _static/figs/xarray-dataset-diagram-new.png
:alt: Xarray data model
:align: center
:width: 500px
:class: dark-light
A better illustration of the variety of Xarray datasets in the wild.
```

## What is an Xarray index?

In a (relational) database, an index is a specific and opaque structure that
allows fast data lookup and retrieval. Likewise, a {py:class}`pandas.Index`
object can be used to efficiently select rows of a {py:class}`pandas.DataFrame`
by one or more labels.

Until recently Xarray exclusively relied on {py:class}`pandas.Index` to allow
fast label-based selection of n-dimensional data via the concept of
{term}`"dimension" coordinates <xarray:Dimension coordinate>`. This approach
worked very well for rectilinear gridded datasets but quickly reached its
limitations when considering other kinds of datasets.

While Xarray still follows the same approach as its default behavior, it has
also become much more flexible: an {py:class}`xarray.Dataset` or
{py:class}`xarray.DataArray` may now have one or more custom
{py:class}`xarray.Index` objects each associated with their own coordinates of
arbitrary dimension(s). Goodbye {term}`"dimension" coordinate
<xarray:Dimension coordinate>` vs. {term}`"non-dimension" coordinate
<xarray:Non-dimension coordinate>` and welcome
{term}`"indexed" coordinate <xarray:Indexed coordinate>` vs.
{term}`"non-indexed" coordinate <xarray:Non-indexed coordinate>`!

{py:class}`xarray.Index` serves a broader purpose than a database index. It
provides an API that allows dealing with coordinate data and metadata in a
highly customizable way for most common Xarray operations (`isel`, `sel`,
`align`, `concat`, `stack`...). Xarray indexes are also stateful objects that
may hold and propagate additional information (arbitrary Python objects) along
with coordinate labels and attributes. {py:class}`xarray.Index` thus represents
a powerful extension mechanism that is complementary to
[accessors](https://docs.xarray.dev/en/stable/internals/extending-xarray.html)
and [IO
backends](https://docs.xarray.dev/en/stable/internals/how-to-add-new-backend.html).


## What is this website?

Xarray flexible indexes unlock a lot of possible use cases. We hope that this
gallery of Xarray built-in and 3rd-party indexes will show a good illustration
of the potential of this feature and will serve as a good reference for
implementing custom indexes (or simply find the existing ones that fulfill your
needs).

```{toctree}
---
caption: Built-in
hidden:
---
builtin/range
builtin/pdinterval
```

```{toctree}
---
caption: Building Blocks
hidden:
---
blocks/transform
blocks/ndpoint
```

```{toctree}
---
caption: Domain Agnostic
hidden:
---
agnostic/pint
```

```{toctree}
---
caption: Earth Sciences
hidden:
---
earth/xproj
earth/raster
earth/xvec
earth/xdggs
earth/forecast
earth/cfvertical
```
