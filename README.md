# xarray-notes
Notes on working with xarray

## Rasters
- rioxarray gives you helpers as a high level interface into rasterio - highly recommended!
- I like the CET Perceptually Uniform colormaps to use here as well https://colorcet.com/download/index.html [also versons for QGIS]

## Rasterisation
- geocube is a good high level api
	- limitation: taps out at integer limit in resolution, have to drop down to something lower level to do that
## Concatenation
- To do lazily, add a dummy dimension
- ``` xr.concat(datasets, dim="dummy").sum("dummy") ```
- https://discourse.pangeo.io/t/tips-for-scalable-spatial-merges-with-many-individual-domains/3658/4

## Proximity
- xarray-spatial - to do all at once distance type functions
