# xarray-notes
Notes on working with xarray

# Concatenation
- To do lazily, add a dummy dimension
- ``` xr.concat(datasets, dim="dummy").sum("dummy") ```
- https://discourse.pangeo.io/t/tips-for-scalable-spatial-merges-with-many-individual-domains/3658/4
