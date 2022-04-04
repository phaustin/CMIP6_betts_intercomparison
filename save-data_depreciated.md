---
jupytext:
  cell_metadata_filter: -all
  notebook_metadata_filter: all,-language_info,-toc,-latex_envs
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

```{code-cell} ipython3
import xarray as xr
import pooch
import pandas as pd
import fsspec
from pathlib import Path
import json
import dask
```

```{code-cell} ipython3
dask.config.set({"array.slicing.split_large_chunks": True})
```

```{code-cell} ipython3
#get esm datastore
odie = pooch.create(
    # Use the default cache folder for the operating system
    path="./.cache",
    base_url="https://storage.googleapis.com/cmip6/",
    # The registry specifies the files that can be fetched
    registry={
        "pangeo-cmip6.csv": "e319cd2bf1daf9b5aa531f92c022d5322ee6bce0b566ac81dfae31dbae203fd9",
    },
)
```

```{code-cell} ipython3
file_path = odie.fetch("pangeo-cmip6.csv")
df_og = pd.read_csv(file_path)
```

```{code-cell} ipython3
fs = fsspec.filesystem("filecache", target_protocol='gs', target_options={'anon': True}, cache_storage='./.cache/files/')
```

```{code-cell} ipython3
#sea area percentage parameters:
lp_var_id = "sftlf" #Percentage of the grid cell occupied by land (including lakes) [%]
lp_monthly_table = "fx" #fixed variables
```

```{code-cell} ipython3
#model parameters:
var_id = "cl"
monthly_table = "Amon"
```

```{code-cell} ipython3
def save_model(mod_id, exp_id, lev_direction):
    if mod_id == "BCC-ESM1":
        lp_exp_id = "1pctCO2"
    else:
        lp_exp_id = "piControl"

    model_path = Path('models/'+mod_id+'_'+exp_id+'.zarr')

    #get the lats and lons for our data:
    query = "variable_id=='"+lp_var_id+"' & experiment_id=='"+lp_exp_id+"' & source_id=='"+mod_id+"' & table_id=='"+lp_monthly_table+"'"
    lp_df = df_og.query(query)
    zstore_url = lp_df["zstore"].values[0]
    the_mapper=fs.get_mapper(zstore_url)
    lp_ds = xr.open_zarr(the_mapper, consolidated=True)
    
    #Cloud Data
    query = "variable_id=='"+var_id+"' & experiment_id=='"+exp_id+"' & source_id=='"+mod_id+"' & table_id=='"+monthly_table+"'"
    cloud_df = df_og.query(query)
    zstore_url = cloud_df["zstore"].values[0]
    the_mapper=fs.get_mapper(zstore_url)
    ds = xr.open_zarr(the_mapper, consolidated=True)
    #print(ds.sizes)
    lp_ds = lp_ds.reindex_like(ds, method="nearest")
    ds_water = ds.where(lp_ds.sftlf == 0.0) #only values over water
    #ds_subset = ds_water.where(ds_water.lev*lev_conversion > 0.7, drop=True)
    ds_sorted = ds_water.sortby(ds_water.lev, ascending=False) #sort lev in descending order
    if lev_direction == "up":
        ds_subset = ds_sorted.isel(lev=slice(-11, -1)) #select 11 smallest levels
    elif lev_direction == "down":
        ds_subset = ds_sorted.isel(lev=slice(0, 11)) #select 11 largest levels
    
    ds_subset = ds_subset.sel(lat=slice(21, 47),  #15-40 degrees North latitude 
                                    lon=slice(200, 243)) #and about 125 to 135 degrees west longitude
    if len(ds_subset.time) > 3000:
        ds_subset = ds_subset.isel(time=slice(0, 3000)) #250 years, so models starting in 1850 will range 1850-2100

    ds_subset.to_zarr(model_path, mode='w')
```

```{code-cell} ipython3
'''
need to fix CESM2 abrupt-4xCO2
'''
#save_model("CanESM5", "historical", 'up')
#save_model("GISS-E2-1-H", "ssp585", 'down')
```

```{code-cell} ipython3
#'''
models_json = open('models.json')
models_dict = json.load(models_json)
```

```{code-cell} ipython3
model_list = ["BCC-ESM1"]
```

```{code-cell} ipython3
for model in models_dict['models']:
    mod_id = model['mod_id']
    if mod_id in model_list:
        exp_id_list = model['exp_id']
        pos_list = model['positive']
        
        for i in range(len(exp_id_list)):
            print('model: ' + mod_id + " exp: " + exp_id_list[i])
            save_model(mod_id, exp_id_list[i], pos_list[i])
#'''
```
