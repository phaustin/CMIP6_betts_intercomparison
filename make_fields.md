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

# Make Fields

Take a domain with mandatory variables and generate the fields required to plot Betts fig 11 

```{code-cell} ipython3
import xarray as xr
import pandas as pd
import numpy as np
import cftime
import matplotlib.pyplot as plt

# Handy metpy tutorial working with xarray:
# https://unidata.github.io/MetPy/latest/tutorials/xarray_tutorial.html#sphx-glr-tutorials-xarray-tutorial-py
import metpy.calc as mpcalc
from metpy.cbook import get_test_data
from metpy.units import units
from metpy.plots import SkewT
```

```{code-cell} ipython3
%run get_domain.ipynb
```

```{code-cell} ipython3
ps = 100000 * units.Pa # temporary hack, should interpolate pressure from daily timeseries (modify `get_domain`)
```

```{code-cell} ipython3
# parse the whole dataset to comform to metpy norms, assign units to everything
dparsed = my_ds.metpy.parse_cf().metpy.quantify()
```

```{code-cell} ipython3
# need to add a step here to select only warm months with PBL development

# take spatial average over domain
spatial_average = dparsed.mean(dim=("lat", "lon"))
```

```{code-cell} ipython3
# separate by soil moisture by rounding to nearest kg/m3 in top soil layer
spatial_average["soil_moisture_grp"] = spatial_average.mrsos.round()[np.isnan(spatial_average.mrsos.values) == False]
```

```{code-cell} ipython3
gbysoil = spatial_average.groupby(spatial_average.soil_moisture_grp)
```

```{code-cell} ipython3
# calculate and plot the average diurnal cycle of lcl height

fig, ax = plt.subplots()
for key in gbysoil.groups.keys():
    # group by hour
    hourly_data = gbysoil[key].groupby(gbysoil[key].time.dt.hour).mean(dim="time") 
    
    # get the extra variables 
    specific_humidity = hourly_data.huss[np.isnan(hourly_data.huss.values) == False]
    surface_temp = hourly_data.tas[np.isnan(hourly_data.tas.values) == False]
    td = mpcalc.dewpoint_from_specific_humidity(ps, surface_temp, specific_humidity)
    
    # find and plot the lcl
    plcl, tlcl = mpcalc.lcl(ps, surface_temp, td)
    ax.plot(plcl, label=key)
    
```

```{code-cell} ipython3

```
