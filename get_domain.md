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
# upload these from a .csv later, containing all applicable models
fetch_this = {"path":None,
              "base_url":"https://storage.googleapis.com/cmip6/",
              "registry":{"pangeo-cmip6.csv": None}
             }

file_params = {"source_id":"GFDL-ESM4", 
               "variable_id":"ps",
               "experiment_id":"piControl",
               "table_id":"Amon"
              }
```

```{code-cell} ipython3
odie = pooch.create(**fetch_this)
```

```{code-cell} ipython3
file_path = odie.fetch("pangeo-cmip6.csv")
df_in = pd.read_csv(file_path)
df_in
```

```{code-cell} ipython3
def fetch_var_exact(the_dict,df_in):
    """
    Extracts a variable given the address dict with keys:
    
        ["source_id", "variable_id", "experiment_id", "table_id"]
    
    returns a dataframe containing the variable
    """
    the_keys = list(the_dict.keys())
    print(the_keys)
    key0 = the_keys[0]
    print(key0)
    print(the_dict[key0])
    hit0 = df_in[key0] == the_dict[key0]
    if len(the_keys) > 1:
        hitnew = hit0
        for key in the_keys[1:]:
            hit = df_in[key] == the_dict[key]
            hitnew = np.logical_and(hitnew,hit)
            print("total hits: ",np.sum(hitnew))
    else:
        hitnew = hit0
    df_result = df_in[hitnew]
    return df_result
        
```

```{code-cell} ipython3
out_df = fetch_var_exact(file_params,df_in) # extract just the field we want
zstore_url = out_df['zstore'].array[0] # save it with zarr
lp_ds = xr.open_zarr(fsspec.get_mapper(zstore_url), consolidated=True)
```

```{code-cell} ipython3

```
