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
import netCDF4 as nc


# Handy metpy tutorial working with xarray:
# https://unidata.github.io/MetPy/latest/tutorials/xarray_tutorial.html#sphx-glr-tutorials-xarray-tutorial-py
import metpy.calc as mpcalc
from metpy.cbook import get_test_data
from metpy.units import units
from metpy.plots import SkewT
```

```{code-cell} ipython3
data_in = xr.open_dataset("data/GFDL-ESM4-piControl", decode_times=False).metpy.quantify()
data_in['time'] = cftime.datetime.fromordinal(data_in.time, calendar='noleap') # manually reconvert to cftime
```

```{code-cell} ipython3

```

```{code-cell} ipython3
ps = 100000 * units.Pa # temporary hack, should interpolate pressure from daily timeseries (modify `get_domain`)
```

```{code-cell} ipython3
specific_humidity = data_in.huss#[np.isnan(data_in.huss.values) == False]#.metpy.quantify()
surface_temp = data_in.tas#[np.isnan(hourly_data.tas.values) == False].metpy.quantify()
data_in["td"] = mpcalc.dewpoint_from_specific_humidity(ps, data_in.tas, data_in.huss)
```

```{code-cell} ipython3
spatial_average = data_in.mean(dim=("lat", "lon"))
spatial_average.time.values # this is super wrong rn
```

```{code-cell} ipython3
# parse the whole dataset to comform to metpy norms, assign units to everything
#dparsed = data_in.metpy.parse_cf().metpy.quantify()
#dparsed
```

```{code-cell} ipython3
# need to add a step here to select only warm months with PBL development. could do:
# summer = my_ds.time[my_ds.time.dt.season == "JJA"]

# take spatial average over domain
#spatial_average = dparsed.mean(dim=("lat", "lon"))
```

```{code-cell} ipython3
# separate by soil moisture by rounding to nearest kg/m3 in top soil layer
spatial_average["soil_moisture_grp"] = spatial_average.mrsos.round()[np.isnan(spatial_average.mrsos.values) == False]
```

```{code-cell} ipython3
gbysoil = spatial_average.groupby(spatial_average.soil_moisture_grp)
gbysoil.groups.keys()
```

```{code-cell} ipython3
gbysoil[8.0].time
```

```{code-cell} ipython3
# calculate and plot the average diurnal cycle of lcl height

fig, ax = plt.subplots()
for key in gbysoil.groups.keys():
    # group by hour
    #hourly_data = gbysoil[key].groupby(gbysoil[key].time.dt.hour).mean(dim="time") 
    hourly_data = gbysoil[key].groupby(gbysoil[key].time.dt.year).mean(dim="time") 
    
    
    # find and plot the lcl
    plcl, tlcl = mpcalc.lcl(ps, hourly_data.tas, hourly_data.td)
    #print(plcl.magnitude)
    ax.plot(plcl[np.isnan(plcl) == False], label=key)
    ax.legend()   
```

```{code-cell} ipython3

```
