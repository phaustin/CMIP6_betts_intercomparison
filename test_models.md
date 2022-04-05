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

```{code-cell} ipython3
hot_models = ['GFDL-ESM4', 'CanESM5', 'HadGEM3-GC31-MM', 'E3SM-1-0']
cold_models = ['INM-CM5-0', 'NorESM2-LM', 'GFDL-ESM4', 'MPI-ESM1-2-HR']
```

```{code-cell} ipython3
for model in hot_models:
    source_id = model
    try:
        %run get_domain.ipynb
    except:
        print(f"{model} does not have required 3hr fields")
```
