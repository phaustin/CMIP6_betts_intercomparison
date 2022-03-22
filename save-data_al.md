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

https://docs.google.com/document/d/1yUx6jr9EdedCOLd--CPdTfGDwEwzPpCF6p1jRmqx-0Q/edit#

https://towardsdatascience.com/a-quick-introduction-to-cmip6-e017127a49d3

https://pcmdi.llnl.gov/CMIP6/Guide/dataUsers.html

http://proj.badc.rl.ac.uk/svn/exarch/CMIP6dreq/tags/latest/dreqPy/docs/CMIP6_MIP_tables.xlsx

https://esgf-node.llnl.gov/search/cmip6/

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
```

```{code-cell} ipython3
file_path = odie.fetch("pangeo-cmip6.csv")
df_og = pd.read_csv(file_path)
```

```{code-cell} ipython3
for var in df_og["variable_id"]:
    print(var)
```

```{code-cell} ipython3
#[print(iden) for iden in df_og.source_id if "CMIP" in str(iden)]
```

```{code-cell} ipython3
def fetch_var_approx(the_dict,df_og):
    the_keys = list(the_dict.keys())
    print(the_keys)
    key0 = the_keys[0]
    print(key0)
    print(the_dict[key0])
    hit0 = df_og[key0].str.find(the_dict[key0]) > -1
    if len(the_keys) > 1:
        hitnew = hit0
        for key in the_keys[1:]:
            hit = df_og[key].str.find(the_dict[key]) > -1
            hitnew = np.logical_and(hitnew,hit)
            print(np.sum(hitnew))
    else:
        hitnew = hit0
    df_result = df_og[hitnew]
    return df_result


def fetch_var_exact(the_dict,df_og):
    the_keys = list(the_dict.keys())
    print(the_keys)
    key0 = the_keys[0]
    print(key0)
    print(the_dict[key0])
    hit0 = df_og[key0] == the_dict[key0]
    if len(the_keys) > 1:
        hitnew = hit0
        for key in the_keys[1:]:
            hit = df_og[key] == the_dict[key]
            hitnew = np.logical_and(hitnew,hit)
            print("total hits: ",np.sum(hitnew))
    else:
        hitnew = hit0
    df_result = df_og[hitnew]
    return df_result
```

```{code-cell} ipython3
# get surface pressure 
ps_dict = dict(source_id = 'GFDL-ESM4', variable_id = 'ps',
                experiment_id = 'piControl', table_id = 'Amon')
local_ps = fetch_var_exact(ps_dict, df_og)
zstore_url = local_ps['zstore'].array[0]
#fs = fsspec.filesystem("filecache", target_protocol='gs', target_options={'anon': True}, cache_storage='/tmp/files/')
#the_mapper = fs.get_mapper(zstore_url)
the_mapper=fsspec.get_mapper(zstore_url)
local_ps = xr.open_zarr(the_mapper, consolidated=True)


# get ap, b variables
cl_dict = dict(source_id = 'GFDL-ESM4', variable_id = 'cl',
                experiment_id = 'piControl', table_id = 'Amon')
local_sig = fetch_var_exact(cl_dict, df_og)
zstore_url = local_sig['zstore'].array[0]
fs = fsspec.filesystem("filecache", target_protocol='gs', target_options={'anon': True}, cache_storage='/tmp/files/')
the_mapper = fs.get_mapper(zstore_url)
local_sig = xr.open_zarr(the_mapper, consolidated=True)
local_sig


# temperature (as a 4d variable template)
ta_dict = dict(source_id = 'GFDL-ESM4', variable_id = 'ta',
                experiment_id = 'piControl', table_id = 'Amon')
local_ta = fetch_var_exact(ta_dict, df_og)
zstore_url = local_ta['zstore'].array[0]
fs = fsspec.filesystem("filecache", target_protocol='gs', target_options={'anon': True}, cache_storage='/tmp/files/')
the_mapper = fs.get_mapper(zstore_url)
local_ta = xr.open_zarr(the_mapper, consolidated=True)
local_ta
```

```{code-cell} ipython3
local_ps.ps
```

```{code-cell} ipython3
press_in = xr.combine_by_coords((local_ps.ps, local_sig.ap, local_sig.b), compat="broadcast_equals")
press_in["press"] = press_in.ap + press_in.b
```

```{code-cell} ipython3
press_in
```

```{code-cell} ipython3
ps_4d = (local_ps.ps.values[-1,...,np.newaxis])
np.shape(ps_4d)
```

```{code-cell} ipython3
ap_4d = (local_sig.ap.values[:,np.newaxis, np.newaxis])
np.shape(ap_4d)
```

```{code-cell} ipython3
ps_4d * ap_4d
```

```{code-cell} ipython3
# select lat/lon domain
domain = ds.sel(lat=slice(10,20), lon=slice(10,21), )

# ap, b extend upwards. psurf has same dims as (lat, lon)
# X = lon, Y = Lat, Z = lev
np.shape(domain.lat)[0]
```

```{code-cell} ipython3
domain
```

```{code-cell} ipython3

```

```{code-cell} ipython3
ap3d = domain.ap.values[:, np.newaxis, np.newaxis]


b3d = domain.b.values[:, np.newaxis, np.newaxis]

domain
```

```{code-cell} ipython3
print(ds.coords['lev'])
ds = ds.sel(lat=slice(32, 46))
ds = ds.sel(lon=slice(200, 243))
print(f"\n\n{ds=}\n\n")
the_cl = ds["cl"]
print(f"\n\n{the_cl=}\n\n")
spatial_mean = ds.mean(dim=["lat", "lon", "lev"])
print(f"\n\n{spatial_mean=}\n\ntype(spatial_mean)\n\n")
times = spatial_mean.indexes["time"]
cloud_cover = spatial_mean['cl']
print(f"{time.time() - t0} elapsed 1")
ds_cloud_cover = cloud_cover.to_dataset()
# ds_cloud_cover.to_netcdf(outfile)
test = cloud_cover[...]
print(f"\n\n{test=},\n{type(test)=}\n\n")
print(f"\n\n{cloud_cover=}\n\n")
#print(cloud_cover)
t1 = time.time()
print(t1-t0);
```

```{code-cell} ipython3
ds
```

```{code-cell} ipython3
ds.b
```
