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
import metpy.calc as mpcalc
from metpy.plots import SkewT
```

```{code-cell} ipython3
# get the domain using 
%run get_domain.ipynb
```

```{code-cell} ipython3
my_ds
```
