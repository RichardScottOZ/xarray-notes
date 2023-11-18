# xarray-notes
Notes on working with xarray

## Rasters
- rioxarray gives you helpers as a high level interface into rasterio - highly recommended!
- I like the CET Perceptually Uniform colormaps to use here as well https://colorcet.com/download/index.html [also versons for QGIS]
- 

## Rasterisation
- geocube is a good high level api
	- limitation: taps out at integer limit in resolution, have to drop down to something lower level to do that
## Concatenation
- To do lazily, add a dummy dimension
- ``` xr.concat(datasets, dim="dummy").sum("dummy") ```
- https://discourse.pangeo.io/t/tips-for-scalable-spatial-merges-with-many-individual-domains/3658/4

## Location Sampling
- Getting data from a grid is often of interest - you can do this in all at once fashion by using xarray to index and return a dataframe
```
def location_sample(gdf, da, name_col):
    """
    Returns a dataframe of points sample from a DataArray by location at once
    
    Args:
        gdf: geodataframe of points
        da: DataArray to sample:
        name_col: String to identify the location names
        
    Returns:
        Onshore location sample dataframe
    
    """

    lat = gdf.geometry.y.tolist()
    lon = gdf.geometry.x.tolist()
    xl = xr.DataArray(lon, dims=['location'],coords={"location":gdf[name_col].tolist()})
    yl = xr.DataArray(lat, dims=['location'],coords={"location":gdf[name_col].tolist()})
    
    dapt = da.sel(x=xl,y=yl,method="nearest")
    dfda = dapt.to_dataframe().reset_index()  
   
    return dfda
```
	
## Proximity
- xarray-spatial - to do all at once distance type functions

## Utility
- A bunch of helper functions I use here and there for speeding up plotting, reading files https://github.com/RichardScottOZ/richardutils/blob/main/src/richardutils/richardutils.py

