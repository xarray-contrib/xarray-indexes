# Xarray Indexes Gallery

## Background

Xarray's data model was initially heavily inspired from the
[NetCDF](https://www.unidata.ucar.edu/software/netcdf/) file format, making it
well suited for working with n-dimensional, rectilinear gridded datasets
commonly found in scientific data analysis, especially in the geosciences. In
fact, Xarray has used many versions of the {ref}`schematic below <xarray-diagram-traditional>` to convey a "canonical" data structure that are
ubiquitous in geosciences (3D datasets with coordinates that are either 2D or
1D).

```{figure} _static/figs/xarray-dataset-diagram-legacy.png
---
name: xarray-diagram-traditional
alt: Xarray data model
align: center
width: 500px
class: dark-light
---
An illustration of the traditional Xarray data model.
```

Over the years, Xarray has evolved and has been adopted in an increasing number
of domains as a convenient, general-purpose Python library for handling
n-dimensional labeled arrays. Xarray's data structures are now being used for
representing a wide range of datasets including sparse data, curvilinear or
irregular grids, staggered grids, discrete global grids, image stacks and vector
data cubes. Consequently, we'll here expand our minds to consider data
structures that are {ref}`much more versatile <xarray-diagram-wild>` 🤯.

```{figure} _static/figs/xarray-dataset-diagram-new.png
---
name: xarray-diagram-wild
alt: Xarray wild dataset
align: center
width: 500px
class: dark-light
---
A better illustration of the variety of Xarray datasets in the wild.
```

## What is an Xarray index?

In order to analyze these increasingly complex data structures in Xarray, we
require a flexible indexing system.

- _What is an index?_

This is a common concept in database systems and data-frame libraries. Generally
speaking:

> An index is a data structure that permits fast data lookup and retrieval.

For example, a {py:class}`pandas.Index` object can be used to efficiently select
rows of a {py:class}`pandas.DataFrame` by one or more labels. Two different
DataFrame objects may be combined together thanks to their index.

- _What about Xarray?_

Until recently Xarray exclusively relied on {py:class}`pandas.Index` to allow
fast label-based selection and alignment of n-dimensional data via the concept
of {term}`"dimension" coordinates <xarray:Dimension coordinate>`. This approach
worked very well for rectilinear gridded datasets but quickly reached its limits
when considering other kinds of datasets.

While Xarray still follows the same approach as its default behavior, it has
also become much more flexible: an {py:class}`xarray.Dataset` or
{py:class}`xarray.DataArray` may now have one or more custom
{py:class}`xarray.Index` objects each associated with their own coordinates of
arbitrary dimension(s). Goodbye {term}`"dimension" coordinate <xarray:Dimension coordinate>` vs. {term}`"non-dimension" coordinate <xarray:Non-dimension coordinate>` and welcome {term}`"index" coordinate <xarray:Indexed coordinate>`
vs. {term}`"non-index" coordinate <xarray:Non-indexed coordinate>`!

- _What is an Xarray index?_

{py:class}`xarray.Index` serves a broader purpose than a database index. It
provides an API that allows dealing with coordinate data and metadata in a
highly customizable way for the most common Xarray operations such as `isel`,
`sel`, `align`, `concat`, `stack`... Xarray indexes usually hold, track and
propagate additional information wrapped in arbitrary Python objects, along with
coordinate labels and attributes. In many cases the propagation of information
via custom indexes is much more efficient and/or reliable than via coordinate
labels and attributes. {py:class}`xarray.Index` thus represents a powerful
extension mechanism that nicely complements
[accessors](https://docs.xarray.dev/en/stable/internals/extending-xarray.html)
and [IO
backends](https://docs.xarray.dev/en/stable/internals/how-to-add-new-backend.html).

## What is this website?

Xarray flexible indexes unlock a lot of possibilities. We hope that this gallery
of Xarray built-in and 3rd-party indexes will show a good illustration of the
potential of this feature and will serve as a good reference for implementing
custom indexes (or simply find the existing ones that fulfill your needs).

## Contribution

Your additions to this gallery are very welcome, particularly for fields outside the Earth Sciences! Please open a pull request on [our GitHub repository](https://github.com/xarray-contrib/xarray-indexes)

```{toctree}
---
caption: Built-in
hidden:
---
builtin/pdindex
builtin/pdmultiindex
builtin/pdrange
builtin/range
builtin/pdinterval
builtin/cfinterval
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
```
