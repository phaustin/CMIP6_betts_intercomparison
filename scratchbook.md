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

# Scratch Book

**Author:** Andrew Loeppky (Lots of code stolen from Jamie Byer)

**Project:** Land-surface-atmosphere coupling - CMIP6 intercomparison 

experiment space for writing, testing code related to Betts CMIP6 intercomparison project

```{code-cell} ipython3
## Part I: Get a CMIP 6 Dataset and Select Domain
```

```{code-cell} ipython3
import xarray as xr
import pooch
import pandas as pd
import fsspec
from pathlib import Path
import time
import numpy as np
import json
import cftime
```

```{code-cell} ipython3
# Attributes of the model we want to analyze (put in csv later)
#source_id = 'CESM2-SE'
source_id = 'GFDL-ESM4'
experiment_id = 'piControl'
#table_id = 'Amon'
table_id = '3hr'

# Domain we wish to study
lats = (10, 20) # lat min, lat max
lons = (20, 29) # lon min, lon max
years = (100, 200)
        #(cftime.DatetimeNoLeap(2000, 1, 1, 1, 30, 0, 0, has_year_zero=True), # start time,
         #cftime.DatetimeNoLeap(2022, 1, 1, 1, 30, 0, 0, has_year_zero=True)) #  end time
ceil = 500 # top of domain, hPa

print(f"""Fetching domain:
          {source_id = }
          {experiment_id = }
          {table_id = }
          {lats = }
          {lons = }
          {years = }
          dataset name: my_ds (xarray Dataset)""")
```

```{code-cell} ipython3
# list of fields required for input calculations
required_fields = ("ps",  # surface pressure
                      "cl",  # cloud fraction
                      "ta",  # air temperature
                      "ts",  # surface temperature
                      "hus", # specific humidity
                      "hfls", # Surface Upward Latent Heat Flux
                      "hfss", # Surface Upward Sensible Heat Flux
                      "rlds",  # surface downwelling longwave
                      "rlus",  # surface upwelling longwave
                      "rsds", # downwelling short wave
                      "rsus", # upwelling short wave
                      "hurs",  # near surface RH
                      "pr", # precipitation, all phases
                      "evspsbl", # evaporation, sublimation, transpiration
                      "wap",  # omega (subsidence rate in pressure coords)
                     )

required_fields = ['tas', 'mrsos', 'mrro', 'tslsi', 'huss'] # temporary hack, but this will work for fig 11
# i need to know which models we intend to parse for this project, they do not all have the same fields
```

```{code-cell} ipython3
# get esm datastore
odie = pooch.create(
    path="./.cache",
    base_url="https://storage.googleapis.com/cmip6/",
    registry={
        "pangeo-cmip6.csv": None
    },
)
file_path = odie.fetch("pangeo-cmip6.csv")
df_in = pd.read_csv(file_path)
```

```{code-cell} ipython3
# extract the names of all fields in our selected model run
available_fields = list(df_in[df_in.source_id == source_id][df_in.experiment_id == experiment_id][df_in.table_id == table_id].variable_id)
```

```{code-cell} ipython3
available_fields
```

```{code-cell} ipython3
# check that our run has all required fields, list problem variables
fields_of_interest = []
missing_fields = []
for rq in required_fields:
    if rq not in available_fields:
        missing_fields.append(rq)
    else:
        fields_of_interest.append(rq)

if missing_fields != []:
    print(f"""WARNING: data from model run:

                {source_id}, 
                {table_id}, 
                {experiment_id} 

         missing required field(s): {missing_fields}""")
```

```{code-cell} ipython3
def fetch_var_exact(the_dict,df_og):
    the_keys = list(the_dict.keys())
    #print(the_keys)
    key0 = the_keys[0]
    #print(key0)
    #print(the_dict[key0])
    hit0 = df_og[key0] == the_dict[key0]
    if len(the_keys) > 1:
        hitnew = hit0
        for key in the_keys[1:]:
            hit = df_og[key] == the_dict[key]
            hitnew = np.logical_and(hitnew,hit)
            #print("total hits: ",np.sum(hitnew))
    else:
        hitnew = hit0
    df_result = df_og[hitnew]
    return df_result
```

```{code-cell} ipython3
def get_field(variable_id, 
              df,
              source_id=source_id,
              experiment_id=experiment_id,
              table_id=table_id):
    """
    extracts a single variable field from the model
    """

    var_dict = dict(source_id = source_id, variable_id = variable_id,
                    experiment_id = experiment_id, table_id = table_id)
    
    local_var = fetch_var_exact(var_dict, df)
    zstore_url = local_var['zstore'].array[0]
    the_mapper=fsspec.get_mapper(zstore_url)
    local_var = xr.open_zarr(the_mapper, consolidated=True)
    return local_var
```

```{code-cell} ipython3
def trim_field(df, lat, lon, years):
    """
    cuts out a specified domain from an xarrray field
    
    lat = (minlat, maxlat)
    lon = (minlon, maxlon)
    """
    new_field = df.sel(lat=slice(lat[0],lat[1]), lon=slice(lon[0],lon[1]))
    new_field = new_field.isel(time=(new_field.time.dt.year > years[0]))
    new_field = new_field.isel(time=(new_field.time.dt.year < years[1]))
    return new_field
```

```{code-cell} ipython3
# grab all fields of interest and combine
my_fields = [get_field(field, df_in) for field in fields_of_interest]
small_fields = [trim_field(field, lats, lons, years) for field in my_fields]
my_ds = xr.combine_by_coords(small_fields, compat="broadcast_equals", combine_attrs="drop_conflicts")
print("Successfully aquired domain")
```

```{code-cell} ipython3
my_ds
```

## Part II: Convert to MetPy Standards and Copy Betts Fig 11

```{code-cell} ipython3
# Handy metpy tutorial working with xarray:
# https://unidata.github.io/MetPy/latest/tutorials/xarray_tutorial.html#sphx-glr-tutorials-xarray-tutorial-py
import metpy.calc as mpcalc
from metpy.cbook import get_test_data
from metpy.units import units
from metpy.plots import SkewT
```

```{code-cell} ipython3
# parse the whole dataset to comform to metpy norms
dparsed = my_ds.metpy.parse_cf()
dparsed
```

```{code-cell} ipython3
dparsed.isel(time=(dparsed.time.dt.year > 20))
```

```{code-cell} ipython3

```
