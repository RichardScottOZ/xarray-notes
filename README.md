# xarray-notes
Notes on working with xarray

## Large Scale Advice
- Pangeo forum : https://discourse.pangeo.io/

## Rasters
- rioxarray gives you helpers as a high level interface into rasterio - highly recommended!
- I like the CET Perceptually Uniform colormaps to use here as well https://colorcet.com/download/index.html [also versons for QGIS]

## 2D
- odc-geo
	- Has some good tools for working in 2D, cogs, dask-backed reprojection, 3 band creation, colorisation and others
	
## 3D
- pyvista-xarray
	- Go from DataArrays to 3D models and vtk	

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
- xarray-spatial - to do all at once distance type functions - requires a little finagling into the dimension format they need

## Utility
- A bunch of helper functions I use here and there for speeding up plotting, reading files https://github.com/RichardScottOZ/richardutils/blob/main/src/richardutils/richardutils.py

## Warped VRT
- To enable out of memory reprojection
```python
dst_crs = CRS.from_epsg(4326)

xres = (right - left) / dst_width
yres = (top - bottom) / dst_height
dst_transform = affine.Affine(xres, 0.0, left,
                              0.0, -yres, top)

vrt_options = {
    'resampling': Resampling.nearest,
    'crs': dst_crs,
    'transform': dst_transform,
    'height': dst_height,
    'width': dst_width,
}

f = r'thermal_endmembers.vrt'
epsg_to = 4326

with rasterio.open(f) as src:
    print('Source CRS:' +str(src.crs))
    with WarpedVRT(src, **vrt_options) as vrt:
        print('Destination CRS:' +str(vrt.crs))
        with ProgressBar():
            thermal_match = rioxarray.open_rasterio(vrt, chunks=(1, 8000,8000))
            thermal_match.rio.to_raster(r'J:\test.tif')
```			

## Zarr
- Possible problems
	- can have problems with unicode - remove encoding
	- problems with zarr chunk encoding - remove encoding
	- problems with zarr chunks between variables - open dataset with set chunks

    ```python
    for v in list(ds.coords.keys()):
        if ds.coords[v].dtype == object:
            ds.coords[v] = ds.coords[v].astype("unicode")

    for v in list(ds.variables.keys()):
        if ds[v].dtype == object:
            ds[v] = ds[v].astype("unicode")
    ```

## Raster padding
- sometimes people pad the edges of rasters with no data - which is of no use for ML
- to fix

```python
wte = tif_dict(r'C:\Users\rscott\dodgytifdir')  # see richardutils
 
for key in wte:
	#nodata value = -100000000.0
	print(key, wte[key].min().values, wte[key].max().values,  wte[key].rio.nodata, wte[key].rio.encoded_nodata)
	# get rid of possible nodata
	wte[key] = wte[key].where(wte[key]  > -100000000.0, drop=True)
	# make any nulls left nodata 
	wte[key] = wte[key].fillna(-100000000.0)
	
	print(key, wte[key].min().values, wte[key].max().values,  wte[key].rio.nodata, wte[key].rio.encoded_nodata)
	da = wte[key]
	da.rio.write_nodata(-100000000.0,inplace=True)
	da.rio.to_raster(r'C:\Users\rscott\Downloads\fixedtifs' + "\\" + key)

    ```