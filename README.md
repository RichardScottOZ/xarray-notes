# xarray-notes
Notes on working with xarray

## Rasters
- rioxarray gives you helpers as a high level interface into rasterio - highly recommended!
## Rasterisation
- geocube is a good high level api
	- limitation: taps out at integer limit in resolution, have to drop down to something lower level to do that
## Concatenation
- To do lazily, add a dummy dimension
- ``` xr.concat(datasets, dim="dummy").sum("dummy") ```
- https://discourse.pangeo.io/t/tips-for-scalable-spatial-merges-with-many-individual-domains/3658/4
