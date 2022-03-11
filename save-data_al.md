---
jupytext:
  cell_metadata_filter: -all
  notebook_metadata_filter: all,-language_info,-toc,-latex_envs
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.13.4
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
        "pangeo-cmip6.csv": "e319cd2bf1daf9b5aa531f92c022d5322ee6bce0b566ac81dfae31dbae203fd9"
    },
)
```

```{code-cell} ipython3
file_path = odie.fetch("pangeo-cmip6.csv")
df_og = pd.read_csv(file_path)
```

```{code-cell} ipython3
df_og.columns
```

```{code-cell} ipython3
[print(iden) for iden in df_og.source_id if "CMIP" in str(iden)]
```

```{code-cell} ipython3
the_dict = dict(source_id = 'GFDL-ESM4', variable_id = 'ps',
                experiment_id = 'piControl', table_id = 'Amon')



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
        
    
out_df = fetch_var_exact(the_dict,df_og)
zstore_url = out_df['zstore'].array[0]
out_df.head()
```

```{code-cell} ipython3
fs = fsspec.filesystem("filecache", target_protocol='gs', target_options={'anon': True}, cache_storage='/tmp/files/')
```

```{code-cell} ipython3
help(fs)
```

```{code-cell} ipython3
dir(fs)
```

```{code-cell} ipython3
the_dict = dict(source_id = 'GFDL-ESM4',variable_id = 'ps',
                experiment_id = 'piControl', table_id = 'Amon')
out_df = fetch_var_exact(the_dict,df_og)
zstore_url_ps = out_df['zstore'].array[0]
```

```{code-cell} ipython3
the_dict = dict(source_id = 'GFDL-ESM4',variable_id = 'cl',
                experiment_id = 'piControl', table_id = 'Amon')
out_df = fetch_var_exact(the_dict,df_og)
zstore_url_cl = out_df['zstore'].array[0]
```

```{code-cell} ipython3
the_mapper = fs.get_mapper(zstore_url)
ds = xr.open_zarr(the_mapper, consolidated=True)
ds.variables['ps']
```

```{code-cell} ipython3
the_dict = dict(source_id = 'GFDL-ESM4',variable_id = 'cl',
                experiment_id = 'piControl', table_id = 'Amon')
out_df = fetch_var_exact(the_dict,df_og)
zstore_url_cl = out_df['zstore'].array[0]
```

```{code-cell} ipython3
the_mapper = fs.get_mapper(zstore_url_cl)
ds = xr.open_zarr(the_mapper, consolidated=True)
ds
```

```{code-cell} ipython3
11*6000*4.*1.e-6
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
