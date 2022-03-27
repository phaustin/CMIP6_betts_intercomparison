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

# Get Domain

**Author:** Andrew Loeppky (Lots of code stolen from Jamie Byer)

**Project:** Land-surface-atmosphere coupling - CMIP6 intercomparison 

This notebook is meant to acquire a dataset from the CMIP6 data library, chop out a pre-specified spatial slice (between coordinates specified by user), and save the dataset in Zarr format. Also adds a 3d pressure field variable, converting from surface pressure and sigma values $ap$ and $b$

## Helpful Docs

https://docs.google.com/document/d/1yUx6jr9EdedCOLd--CPdTfGDwEwzPpCF6p1jRmqx-0Q/edit#

https://towardsdatascience.com/a-quick-introduction-to-cmip6-e017127a49d3

https://pcmdi.llnl.gov/CMIP6/Guide/dataUsers.html

http://proj.badc.rl.ac.uk/svn/exarch/CMIP6dreq/tags/latest/dreqPy/docs/CMIP6_MIP_tables.xlsx

https://esgf-node.llnl.gov/search/cmip6/

```{code-cell} ipython3
# Attributes of the model we want to analyze (put in csv later)
source_id = 'GFDL-ESM4'
experiment_id = 'piControl'
table_id = 'Amon'

# Domain we wish to study
lats = (10, 20) # lat min, lat max
lons = (20, 29) # lon min, lon max
ceil = 500 # top of domain, hPa

# variables of interest
fields_of_interest = ("ps",  # surface pressure
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
                      "wap"  # omega (subsidence rate in pressure coords)
                     )
                     
                        #"mrsol" # total soil water content 
                     #"bldep", ) # boundary layer depth
                     # "tsl")  # soil temperature
                      #"mrsos",) # soil water content, top 10cm
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
```

```{code-cell} ipython3
#get esm datastore
odie = pooch.create(
    path="./.cache",
    base_url="https://storage.googleapis.com/cmip6/",
    registry={
        "pangeo-cmip6.csv": None
    },
)
file_path = odie.fetch("pangeo-cmip6.csv")
df_og = pd.read_csv(file_path)
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
def trim_field(df, lat, lon):
    """
    cuts out a specified domain from an xarrray field
    
    lat = (minlat, maxlat)
    lon = (minlon, maxlon)
    """
    new_field = df.sel(lat=slice(lat[0],lat[1]), lon=slice(lon[0],lon[1]))
    return new_field
```

## Create one big dataset to represent our domain

```{code-cell} ipython3
# grab all fields of interest and combine
my_fields = [get_field(field, df_og) for field in fields_of_interest]
small_fields = [trim_field(field, lats, lons) for field in my_fields]
my_ds = xr.combine_by_coords(small_fields, compat="broadcast_equals", combine_attrs="drop_conflicts")
```

```{code-cell} ipython3
# add pressure field, convert from sigma pressure
def press_from_sigma(ds):
    """
    takes in an xarray Dataset with variables:
        "ps" - surface pressure (Pa)
        "ap" - sigma pressure coordinate
        "b"  - sigma pressure coordinate

    returns the Dataset with a variable "p", the full pressure (Pa)
    """
    ds["p"] = ds.ap + ds.b * ds.ps
    return ds
```

```{code-cell} ipython3
def p_lcl(ds):
    """
    takes in an xarray Dataset with variables:
        "p" - pressure
        "ts" - surface (2m) air temperature
        "hurs" - near surface relative humidity
        "hus" - specific humidity
        
    horizontally averaged over the domain of the Dataset
    """
    pass
```

```{code-cell} ipython3
#mean_tsurf = my_ds.ts.mean(dim=("lat", "lon"))
#mean_psurf = my_ds.ps.mean(dim=("lat", "lon"))
```

```{code-cell} ipython3
# apply functions defined above to the field
my_ds = press_from_sigma(my_ds)
```

```{code-cell} ipython3
#my_ds["hus"]
```

```{code-cell} ipython3
print(f"""Fetched domain:
          {source_id = }
          {experiment_id = }
          {table_id = }
          {lats = }
          {lons = }
          dataset name: my_ds (xarray Dataset)""")
```
