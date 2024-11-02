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
 - GTI Driver
 	- New in GDAL 3.9 - allows on the fly lazy tile indexing with a vector dataset - try FlatGeoBuf - with a location of the file and a bounding box for each raster in the CRS
   	- Handles any number of files or projections
	
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

```python
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

## long name handling
- if bands have names and you wanted to output them, here's a one-liner
    - image = image.rename({band:image[band].attrs["long_name"] for band in image})    https://github.com/corteva/rioxarray/issues/736


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

## Naive scipy gridding
```python
#X = np.linspace(bb[0],bb[2],1000)
#Y = np.linspace(bb[1],bb[3],1000)

#xi = gdf["X3107"].to_numpy()
#yi = gdf["Y3107"].to_numpy()
#zi = gdf["Z"].to_numpy()
#sgi = gdf["OTHER_THING_TO_GRID"].to_numpy()
#srcwidth = 1000
#srcheight = 1000

gdfSB 

bb = gdfSB.total_bounds

srcwidth = int(width / 200)
srcheight = int(height / 200)

X = np.linspace(bb[0],bb[2],srcwidth)
Y = np.linspace(bb[1],bb[3],srcheight)
#Z = np.linspace(z0, altitude,srcdepth)  #need an altitude
#X, Y, Z = np.meshgrid(X, Y, Z)
X, Y = np.meshgrid(X, Y)
#print(X.shape, Y.shape, Z.shape)
print(X.shape, Y.shape)

points = np.array([gdfSB.geometry.x.to_list(),gdfSB.geometry.y.to_list()])
v = gdfSB['THING_TO_GRID'].to_list()
#vinterp = griddata(points.T, v, (X, Y, Z), method='nearest')

vinterp = griddata(points.T, v, (X, Y), method='linear')  ## use nearest for other uses,e tc
print(vinterp.shape)

## interpolating for padding
- https://docs.xarray.dev/en/stable/generated/xarray.DataArray.interpolate_na.html

## geocube - rasterisation of polygons - dask backed
- https://github.com/corteva/geocube/issues/41
```python
def make_geocube_like_dask2(
        df: gpd.GeoDataFrame,
        measurements: Optional[List[str]],
        like: xr.core.dataarray.DataArray,
        fill: int=0,
        rasterize_function:callable=partial(geocube.rasterize.rasterize_image, all_touched=True),
        **kwargs
):
    def rasterize_block(block):
        return(
            make_geocube(
                df,
                measurements=measurements,
                like=block,
                fill=fill,
                rasterize_function=rasterize_function,
            )
            .to_array(measurements[0])
            .assign_coords(block.coords)
        )

    like = like.rename(dict(zip(['band'], measurements)))
    return like.map_blocks(
        rasterize_block,
        template=like
    )
```

## lazy concat trick
- nd = xr.concat(newlist, dim='dummy').mean(dim='dummy',keep_attrs=True)

## pangeo
- xbatcher cloud streaming talk Joe Hamman https://www.youtube.com/watch?v=HYvhj9W2qHQ&t=661s

## Errors

- ValueError: zero-size array to reduction operation maximum which has no identity
    - a zero dimension this means, which is not too useful - so needs to be fixed or bypassed in logic

## vectorised sklearn
- https://gist.github.com/RichardScottOZ/89e054ca5c4c9fbbbf8d5d6499df856c 

## ESRI
- gdal can deal with terrible GDB rasters
- gemgis has a builtin https://github.com/cgre-aachen/gemgis/issues/302
- https://github.com/cgre-aachen/gemgis/blob/main/gemgis/raster.py

 ```python
import gemgis
from gemgis.raster import read_raster_gdb
read_raster_gdb('crappy_raster_database_here.gdb')

#this exports all the raster subdatasets for you



```


## Attribute issues
```python
vars_list = list(xr_df.data_vars)  
for var in vars_list:  
    del xr_df[var].attrs['grid_mapping']
```

## Fix backwards y-coords for a raster
```python
structures = structures.assign_coords(y=structures.y * -1)
structures_age_confid = structures_age_confid.assign_coords(y=structures_age_confid.y * -1)
# fix the transform
from affine import Affine
new_transform = Affine(0.016666666666666666, 0.0, -180.0, 0.0, -0.016666666666666666, 89.98333333333332)

or another case

tt = tt.isel(y=slice(None, None, -1))


```

## Fix 360 degree xcoordinate problems that have already translated to -180 and 180 incorrectly
```python
def fix_x(test2, strdim):
    test2 = test2.transpose(strdim,"y","x")
    test2['x'] = test2['x'] - 180
    test2['x'] = test2['x'].where(test2['x'] > -180, other=test2['x'] + 360)
    test2 = test2.sortby('x')
    
    return test2
```

### pygplates/gplately
```python
rasterfile = 'somedirectory/pasthistory_data.tif'

print("RASTERFILE:",rasterfile)
da = rioxarray.open_rasterio(rasterfile)

use_model = gplately.PlateReconstruction(rot_file)
#extent = [ da.rio.bounds()[0],da.rio.bounds()[2],da.rio.bounds()[1],da.rio.bounds()[3] ]
extent = [ da.rio.bounds()[0],da.rio.bounds()[2],da.rio.bounds()[3],da.rio.bounds()[1] ]
#extent = [-180,180,-90,90]
print("EXTENT:",extent)

print(da)

raster = gplately.Raster(da.variable.data[0],
			  plate_reconstruction=use_model, 
			  extent=extent,
			  time=time)
    
print("PLATE POLYS:", plate_polys)

print("RECONSTRUCTING: ",time,'present_day' + "\\" + tif)
reverse_reconstructed = raster.reconstruct(0, fill_value=np.nan, partitioning_features=plate_polys)

dapresent = copy.deepcopy(da)

dapresent.values = reverse_reconstructed.data.reshape(dapresent.values.shape[0], dapresent.values.shape[1],dapresent.values.shape[2])

print(dapresent)

print("WRITING:",'present_day' + "\\" + tif)
#dapresent = dapresent.astype('float32')
dapresent.rio.to_raster('present_day' + "\\" + tif)
```

## hdf5 errors

- [Errno -101] NetCDF: HDF error: '/home/rscott/data/STELLAR-BHP/SeaFloorAgeGrids-MantleFrame/agegrid-0.nc'

- use
- import netCDF4  at the start before anything else uses HDF


