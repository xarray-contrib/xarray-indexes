---
jupytext:
  text_representation:
    format_name: myst
kernelspec:
  display_name: Python 3
  name: python
---

# Discrete Global Grid Systems with `xdggs.DGGSIndex`

````{grid}
```{grid-item}
:columns: 3
```{image} https://xdggs.readthedocs.io/en/latest/_static/xdggs_logo.png
---
alt: xdggs logo
width: 200px
align: center
class: dark-light
---
```
```{grid-item}
:columns: 9
```{seealso}
Learn more at the [xdggs](https://xdggs.readthedocs.io/en/latest/) documentation page.
```
````

## Highlights

xdggs provides a unified API for interacting with various Discrete Global Grid Systems (DGGS), like [HEALPix](https://healpix.sourceforge.io/html/intro.htm) or [H3](https://h3geo.org). It does so by parsing out grid metadata from the spatial coordinate containing the cell ids and persisting it on a "meta-index", an index that wraps another index (most commonly a `pandas` index).

```{figure} h3.png
The H3 grid, a hexagonal DGGS designed for navigation on land.
```

## Example

Here's a dataset on a HEALPix grid:

```{code-cell} python
import xdggs

ds = xdggs.tutorial.open_dataset("air_temperature", "healpix")
ds
```

with that, we can decode the metadata:

```{code-cell} python
decoded = ds.dggs.decode()
decoded
```

Note how the `cell_ids` coordinate is now associated with a `HealpixIndex`? This makes it harder to lose the parsed information, and thus having to parse it again.

We can also have a look at the parsed metadata:

```{code-cell} python
decoded.dggs.grid_info
```

Here's how the dataset looks:

```{code-cell} python
decoded["air"].isel(time=0).dggs.explore(alpha=0.8)
```

In a future version of `xdggs` this may also support more operations, like alignment, selection using parent cell ids, or lazy coordinates for cell centers / cell boundaries.
