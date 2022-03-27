---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.13.7
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# Make Figures

**Author:** Andrew Loeppky
    
**Project:** Land-surface-atmosphere coupling - CMIP6 intercomparison 

This notebook recreates figures 10, 11 from Betts (2009), and generate a skew-T plot averaged over the domain selected in `get_domain.ipynb`. 

```{code-cell} ipython3
import matplotlib.pyplot as plt
import numpy as np
import xarray as xr

# Handy metpy tutorial working with xarray:
# https://unidata.github.io/MetPy/latest/tutorials/xarray_tutorial.html#sphx-glr-tutorials-xarray-tutorial-py
import metpy.calc as mpcalc
from metpy.cbook import get_test_data
from metpy.units import units
from metpy.plots import SkewT
```

```{code-cell} ipython3
# get the domain using 
%run get_domain.ipynb
```

```{code-cell} ipython3

```

```{code-cell} ipython3
tsurf = my_ds.ts
```

```{code-cell} ipython3
tsurf.metpy.time
```

```{code-cell} ipython3
x, y = tsurf.metpy.coordinates("x", "y")
x
```

```{code-cell} ipython3
dparsed = my_ds.metpy.parse_cf()
dparsed
```

```{code-cell} ipython3
dparsed["td"] = mpcalc.dewpoint_from_relative_humidity(dparsed.ts, dparsed.hurs)
dparsed
```

```{code-cell} ipython3
ps_mean = (dparsed.ps.mean(dim=("lon", "lat"))).values
ts_mean = (dparsed.ts.mean(dim=("lon", "lat"))).values
td_mean = (dparsed.td.mean(dim=("lon", "lat"))).values
#tlcl, plcl = mpcalc.lcl(ps_mean, ts_mean, td_mean)
ps_mean
```

```{code-cell} ipython3
td_mean
```

```{code-cell} ipython3
plcl, tlcl = mpcalc.lcl(ps_mean * units.pascal, ts_mean * units.kelvin, td_mean * units.kelvin)
```

```{code-cell} ipython3
plt.plot(plcl)
```
