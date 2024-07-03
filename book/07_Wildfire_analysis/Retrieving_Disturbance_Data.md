---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.16.1
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

# Retrieving Disturbance Data

The [OPERA DIST-HLS data product](https://lpdaac.usgs.gov/documents/1766/OPERA_DIST_HLS_Product_Specification_V1.pdf) can be used to study the impacts and evolution of wildfires at a large scale. In this notebook, we will retrieve data associated with the [2023 Greece wildfires](https://en.wikipedia.org/wiki/2023_Greece_wildfires) to understand its evolution and extent. We will also generate a time series visualization of the event.

In particular, we will be examining the area around the city of [Alexandroupolis](https://en.wikipedia.org/wiki/Alexandroupolis) which was severely impacted by the wildfires, resulting in loss of lives, property, and forested areas.

```python
# Plotting imports
import matplotlib.pyplot as plt
from matplotlib.colors import ListedColormap
from rasterio.plot import show
from rasterio.warp import transform_bounds
from mpl_toolkits.axes_grid1.anchored_artists import AnchoredSizeBar
import geoviews as gv
from geoviews import opts
from rasterio.crs import CRS
gv.extension('bokeh')

# GIS imports
from shapely.geometry import Point
from osgeo import gdal
from rasterio.merge import merge
import rasterio
import contextily as cx
import rioxarray
import xyzservices.providers as xyz
import cartopy.crs as ccrs 

# data wrangling imports
import pandas as pd
import numpy as np
import hvplot.xarray  # noqa
import xarray as xr

# misc imports
from datetime import datetime, timedelta
from collections import defaultdict

# STAC imports to retrieve cloud data
from pystac_client import Client

# GDAL setup for accessing cloud data
gdal.SetConfigOption('GDAL_HTTP_COOKIEFILE','~/cookies.txt')
gdal.SetConfigOption('GDAL_HTTP_COOKIEJAR', '~/cookies.txt')
gdal.SetConfigOption('GDAL_DISABLE_READDIR_ON_OPEN','EMPTY_DIR')
gdal.SetConfigOption('CPL_VSIL_CURL_ALLOWED_EXTENSIONS','TIF, TIFF')
```

### Deforestation in Madre de Dios, Peru

```python
madre_de_dios = (-70.61, -11.89)

ref_crs = CRS.from_epsg(4326)
dst_crs = CRS.from_epsg(3857)
map_bounds = transform_bounds(ref_crs, dst_crs, *Point(*madre_de_dios).buffer(10).bounds)
```

```python
madre_de_dios_gv = gv.Points([madre_de_dios])

basemap = gv.tile_sources.OSM
plot = (madre_de_dios_gv*basemap).opts(
    opts.Points(
        color='red',
        alpha=0.75,
        size=25,
        width=800,
        height=800,
        xlim=(map_bounds[0], map_bounds[2]),
        ylim=(map_bounds[1], map_bounds[3]))
)
plot
```

```python
# We will search data through the product record
start_date = datetime(year=2022, month=1, day=1)
stop_date = datetime.now()

# We open a client instance to search for data, and retrieve relevant data records
STAC_URL = 'https://cmr.earthdata.nasa.gov/stac'

# Setup PySTAC client
# LPCLOUD refers to the LP DAAC cloud environment that hosts earth observation data
catalog = Client.open(f'{STAC_URL}/LPCLOUD/') 

collections = ["OPERA_L3_DIST-ALERT-HLS_V1"]

# We would like to search data for August-September 2023
date_range = f'{start_date.strftime("%Y-%m-%d")}/{stop_date.strftime("%Y-%m-%d")}'

search_opts = {
    'bbox' : Point(*madre_de_dios).buffer(.1).bounds, 
    'collections': collections,
    'datetime' : date_range,
}

search = catalog.search(**search_opts)
results = list(search.items_as_dicts())
print(f"Number of tiles found intersecting given AOI: {len(results)}")
```

```python
# Load results into a pandas dataframe
granules = search_to_df(results)

# filter to study a single geographic area
granules = granules[granules.tile_id == 'T19LCG']

# load data into an xarray dataset
dataset= urls_to_dataset(granules[::10])
```

```python
# Define color map to generate plot (Red, Green, Blue, Alpha)
COLORS = [(1, 1, 1, 0)] * 256  # Initial set all values to white, with zero opacity
COLORS[6:9] = [(1, 0, 0, 1)]       # Set class 6 to Red with 100% opacity

# Create a ListedColormap
cmap = ListedColormap(colors)
```

```python
img = dataset.hvplot.quadmesh(title = 'Deforestation in Madre de Dios. Disturbance Alerts',
                            x='lon', y='lat', 
                            project=True, rasterize=True, 
                            cmap=COLORS, 
                            colorbar=False,
                            widget_location='bottom',
                            tiles = gv.tile_sources.OSM,
                            xlabel='Longitude (degrees)',ylabel='Latitude (degrees)',
                            fontscale=1.25,
                            frame_width=900, frame_height=900,)

img
```

### 2023 Greece wildfires

```python
# Define AOI as a point, to which we will apply a buffer
dadia_forest = (26.18, 41.08)

ref_crs = CRS.from_epsg(4326)
dst_crs = CRS.from_epsg(3857)
map_bounds = transform_bounds(ref_crs, dst_crs, *Point(*dadia_forest).buffer(.5).bounds)
```

```python
dadia_forest_gv = gv.Points([dadia_forest])

basemap = gv.tile_sources.OSM
plot = (dadia_forest_gv*basemap).opts(
    opts.Points(
        color='red',
        alpha=0.75,
        size=25,
        width=800,
        height=800,
        xlim=(map_bounds[0], map_bounds[2]),
        ylim=(map_bounds[1], map_bounds[3]))
)
plot
```

```python
# Define data search parameters

# We will search data for the month of March 2024
start_date = datetime(year=2023, month=8, day=1)
stop_date = datetime(year=2023, month=9, day=30)

# We open a client instance to search for data, and retrieve relevant data records
STAC_URL = 'https://cmr.earthdata.nasa.gov/stac'

# Setup PySTAC client
# LPCLOUD refers to the LP DAAC cloud environment that hosts earth observation data
catalog = Client.open(f'{STAC_URL}/LPCLOUD/') 

collections = ["OPERA_L3_DIST-ALERT-HLS_V1"]

# We would like to search data for August-September 2023
date_range = f'{start_date.strftime("%Y-%m-%d")}/{stop_date.strftime("%Y-%m-%d")}'

search_opts = {
    'bbox' : Point(*dadia_forest).buffer(0.1).bounds, 
    'collections': collections,
    'datetime' : date_range,
}

search = catalog.search(**search_opts)
```

NOTE: The OPERA DIST data product is hosted on [LP DAAC](https://lpdaac.usgs.gov/news/lp-daac-releases-opera-land-surface-disturbance-alert-version-1-data-product/), and this is specified when setting up the PySTAC client to search their catalog of data products in the above code cell.

```python
results = list(search.items_as_dicts())
print(f"Number of tiles found intersecting given AOI: {len(results)}")
```

Let's load the search results into a pandas dataframe

```python
def search_to_df(results, layer_name = 'VEG-DIST-STATUS'):

        times = pd.DatetimeIndex([result['properties']['datetime'] for result in results]) # parse of timestamp for each result
        data = {'hrefs': [value['href'] for result in results for key, value in result['assets'].items() if layer_name in key],  # parse out links only to DIST-STATUS data layer
                'tile_id': [value['href'].split('/')[-1].split('_')[3] for result in results for key, value in result['assets'].items() if layer_name in key]}

        # Construct pandas dataframe to summarize granules from search results
        granules = pd.DataFrame(index=times, data=data)
        granules.index.name = 'times'

        return granules
```

```python
granules = search_to_df(results)
granules.head()

```

```python
# Let's refine the dataframe a bit more so that we group together granules by 
# date of acquisition - we don't mind if they were acquired at different times 
# of the same day

refined_granules = defaultdict(list)

for i, row in granules.iterrows():
    refined_granules[i.strftime('%Y-%m-%d')].append(row.hrefs)

refined_granules = pd.DataFrame(index=refined_granules.keys(), data = {'hrefs':refined_granules.values()})
```

```python
# The wildfire near Alexandroupolis started on August 21st and rapidly spread, particularly affecting the nearby Dadia Forest
# For demonstration purposes, let's look at three dates to study the extent of the fire - 
# August 1st, August 25th, and September 19th
# We will plot the OPERA-DIST-ALERT data product, highlighting only those pixels corresponding to confirmed vegetation damage,
# and in particular only those pixels where at least 50% of the area was affected (layer value 6)

dates_of_interest = [datetime(year=2023, month=8, day=1), datetime(year=2023, month=8, day=26), datetime(year=2023, month=9, day=18)]
hrefs_of_interest = [x.hrefs for i, x in refined_granules.iterrows() if datetime.strptime(i, '%Y-%m-%d') in dates_of_interest]
```

**Relevant layer Values for DIST-ALERT product:**

* **0:** No disturbance<br>
* **1:** First detection of disturbance with vegetation cover change <50% <br>
* **2:** Provisional detection of disturbance with vegetation cover change <50% <br>
* **3:** Confirmed detection of disturbance with vegetation cover change <50% <br>
* **4:** First detection of disturbance with vegetation cover change >50% <br>
* **5:** Provisional detection of disturbance with vegetation cover change >50% <br>
* **6:** Confirmed detection of disturbance with vegetation cover change >50% <br>

```python
# Define color map to generate plot (Red, Green, Blue, Alpha)
colors = [(1, 1, 1, 0)] * 256  # Initial set all values to white, with zero opacity
colors[6] = (1, 0, 0, 1)       # Set class 6 to Red with 100% opacity

# Create a ListedColormap
cmap = ListedColormap(colors)
```

```python
fig, ax = plt.subplots(1, 3, figsize = (30, 10))
crs = None

for i, (date, hrefs) in enumerate(zip(dates_of_interest, hrefs_of_interest)):
    
    # Read the crs to be used to generate basemaps
    if crs is None:
        with rasterio.open(hrefs[0]) as ds:
            crs = ds.crs

    if len(hrefs) == 1:
        with rasterio.open(hrefs[0]) as ds:
            raster = ds.read()
            transform = ds.transform
    else:
        raster, transform = merge(hrefs)

    show(raster, ax=ax[i], transform=transform, interpolation='none')
    cx.add_basemap(ax[i], crs=crs, zoom=9, source=cx.providers.OpenStreetMap.Mapnik)
    show(raster, ax=ax[i], transform=transform, interpolation='none', cmap=cmap)

    scalebar = AnchoredSizeBar(ax[i].transData,
                            10000 , '10 km', 'lower right', 
                            color='black',
                            frameon=False,
                            pad = 0.25,
                            sep=5,
                            fontproperties = {'weight':'semibold', 'size':12},
                            size_vertical=300)

    ax[i].add_artist(scalebar)
    ax[i].ticklabel_format(axis='both', style='scientific',scilimits=(0,0),useOffset=False,useMathText=True)
    ax[i].set_xlabel('UTM easting (meters)')
    ax[i].set_ylabel('UTM northing (meters)')
    ax[i].set_title(f"Disturbance extent on: {date.strftime('%Y-%m-%d')}")
```

Next, let's calculate the area extent of damage over time

```python
damage_area = []
conversion_factor = (30*1e-3)**2 # to convert pixel count to area in km^2; each pixel is 30x30 meters

# this will take a few minutes to run, since we are retrieving data for multiple days
for index, row in refined_granules.iterrows():
    raster, transform = merge(row.hrefs)
    damage_area.append(np.sum(raster==6)*conversion_factor)

refined_granules['damage_area'] = damage_area

```

```python
fig, ax = plt.subplots(1, 1, figsize=(20, 10))
ax.plot([datetime.strptime(i, '%Y-%m-%d') for i in refined_granules.index], refined_granules['damage_area'], color='red')
ax.grid()
plt.ylabel('Area damaged by wildfire (km$^2$)', size=15)
plt.xlabel('Date', size=15)
plt.xticks([datetime(year=2023, month=8, day=1) + timedelta(days=6*i) for i in range(11)], size=14)
plt.title('2023 Dadia forest wildfire detected extent', size=14)
```

### Great Green Wall, Sahel Region, Africa

```python
ndiaye_senegal = (-16.0913, 16.528)

ref_crs = CRS.from_epsg(4326)
dst_crs = CRS.from_epsg(3857)
map_bounds = transform_bounds(ref_crs, dst_crs, *Point(*ndiaye_senegal).buffer(5).bounds)
```

```python
print(gv.tile_sources.OSM)
```

```python
ndiaye_senegal_gv = gv.Points([ndiaye_senegal])

basemap = gv.tile_sources.OSM
plot = (ndiaye_senegal_gv*basemap).opts(
    opts.Points(
        color='red',
        alpha=0.75,
        size=25,
        width=800,
        height=800,
        xlim=(map_bounds[0], map_bounds[2]),
        ylim=(map_bounds[1], map_bounds[3]))
)
plot
```

```python
# We will search data through the product record
start_date = datetime(year=2022, month=1, day=1)
stop_date = datetime.now()

# We open a client instance to search for data, and retrieve relevant data records
STAC_URL = 'https://cmr.earthdata.nasa.gov/stac'

# Setup PySTAC client
# LPCLOUD refers to the LP DAAC cloud environment that hosts earth observation data
catalog = Client.open(f'{STAC_URL}/LPCLOUD/') 

collections = ["OPERA_L3_DIST-ALERT-HLS_V1"]

# We would like to search data for August-September 2023
date_range = f'{start_date.strftime("%Y-%m-%d")}/{stop_date.strftime("%Y-%m-%d")}'

search_opts = {
    'bbox' : Point(*ndiaye_senegal).buffer(.1).bounds, 
    'collections': collections,
    'datetime' : date_range,
}

search = catalog.search(**search_opts)
results = list(search.items_as_dicts())
print(f"Number of tiles found intersecting given AOI: {len(results)}")
```

```python
granules = search_to_df(results)
```

```python
def urls_to_dataset(granule_dataframe):
    '''method that takes in a list of OPERA tile URLs and returns an xarray dataset with dimensions
    latitude, longitude and time'''

    dataset_list = []
    
    for i, row in granule_dataframe.iterrows():
        with rasterio.open(row.hrefs) as ds:
            # extract CRS string
            crs = str(ds.crs).split(':')[-1]

            # extract the image spatial extent (xmin, ymin, xmax, ymax)
            xmin, ymin, xmax, ymax = ds.bounds

            # the x and y resolution of the image is available in image metadata
            x_res = np.abs(ds.transform[0])
            y_res = np.abs(ds.transform[4])

            # read the data 
            img = ds.read()

            # Ensure img has three dimensions (bands, y, x)
            if img.ndim == 2:
                img = np.expand_dims(img, axis=0) 

            

        lon = np.arange(xmin, xmax, x_res)
        lat = np.arange(ymax, ymin, -y_res)

        lon_grid, lat_grid = np.meshgrid(lon, lat)

        da = xr.DataArray(
            data=img,
            dims=["band", "y", "x"],
            coords=dict(
                lon=(["y", "x"], lon_grid),
                lat=(["y", "x"], lat_grid),
                time=i,
                band=np.arange(img.shape[0])
            ),
            attrs=dict(
                description="OPERA DIST data",
                units=None,
            ),
        )
        da.rio.write_crs(crs, inplace=True)

        dataset_list.append(da)
    return xr.concat(dataset_list, dim='time').squeeze()

# since there are a large number of results, we pick only every 100th tile this demonstration
dataset= urls_to_dataset(granules[::10])
```

```python
COLORS = [(0, 0, 0, 0)]*256
for i in [5, 6, 8]:
    COLORS[i] = (255, 0, 0, 1)
```

```python
img = dataset.hvplot.quadmesh(title = 'Great Green Wall, Sahel Region, Africa. Disturbance Alerts',
                            x='lon', y='lat', 
                            project=True, rasterize=True, 
                            cmap=COLORS, 
                            colorbar=False,
                            widget_location='bottom',
                            tiles = gv.tile_sources.OSM,
                            xlabel='Longitude (degrees)',ylabel='Latitude (degrees)',
                            fontscale=1.25,
                            frame_width=800, frame_height=800,)

img
```

### Deforestation in Parra, Brazil


Deforestation in the Amazon Rainforest of [Para, Brazil](https://storymaps.arcgis.com/stories/4036b210530b47e884befbc35ec7c482) is an ongoing problem with significant environmental impacts as well displacement of local indigenous populations. The tree loss is driven mainly by forest being cleared for cattle ranching, soy cultivation, and illegal gold mining and logging.

```python
maravilha_lonlat = (-60.732, -2.403)

ref_crs = CRS.from_epsg(4326)
dst_crs = CRS.from_epsg(3857)
map_bounds = transform_bounds(ref_crs, dst_crs, *Point(*maravilha_lonlat).buffer(10).bounds)
```

```python
maravilha = gv.Points([maravilha_lonlat])

basemap = gv.tile_sources.OSM
plot = (maravilha*basemap).opts(
    opts.Points(
        color='red',
        alpha=0.75,
        size=25,
        width=800,
        height=800,
        xlim=(map_bounds[0], map_bounds[2]),
        ylim=(map_bounds[1], map_bounds[3]))
)
plot
```
