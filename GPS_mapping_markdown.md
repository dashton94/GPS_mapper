# GPS Mapping: the Strava API, folium and lidar

As a keen runner and data enthusiast, I have developed a script to extract GPS data from my runs which are uploaded to the popular fitness-focused social media app, Strava, before overlaying these data on maps which use the python package, folium, in order to recreate a fully interactive, global heatmap of my gps recorded runs. Beyond this, I explore the gpx code of a single activity to highlight key features of the terrain, before plotting this in 3D and utilising Lidar mapping of the Isle of Wight to project this onto the measured terrain.

In this notebook I set show the workings of this setup step-by-step. See the associated markdown file for full visualisations of this work.


```python
%matplotlib inline
```


```python
import time
import os

import pandas as pd
import folium
from tqdm import tqdm

from stravaio import strava_oauth2
from stravalib import Client
from dotenv import load_dotenv
```

## Access the Strava API

The Strava API may be utilised following the instructions set out in https://developers.strava.com/, and developers are able to create their own API application. This API app gives the user both a Client ID and Secret ID. These may be used to access the personal gps data uploaded to the service. Here, I have stored my ID information in a .env file, which I later load in using os.getenv().

The .env file takes this form, with 'XXX' and 'YYY' being a string of numbers/letters given by the strava API:
MY_STRAVA_CLIENT_ID = 'XXX'
MY_STRAVA_CLIENT_SECRET = 'YYY'

The code below extracts these activities into a pandas dataframe with defined columns of interest.


```python
if load_dotenv() == False:  # Check if local .env file exists (with personal IDs)
    print("Create local .env file in the format written above")
    raise OSError

MY_STRAVA_CLIENT_ID = os.getenv("MY_STRAVA_CLIENT_ID")  # Input personal ID
MY_STRAVA_CLIENT_SECRET = os.getenv("MY_STRAVA_CLIENT_SECRET")  # Input personal secret ID

out = strava_oauth2(client_id=MY_STRAVA_CLIENT_ID, client_secret=MY_STRAVA_CLIENT_SECRET)
MY_STRAVA_ACCESS_TOKEN = out['access_token']

client = Client(access_token=MY_STRAVA_ACCESS_TOKEN)  # Use the Client function in Strava's API

df = pd.DataFrame()  # Initialise a dataframe
activities = client.get_activities()  # Extract activities

my_cols =['name', # Define column names for the df as appropriate
          'start_date_local',
          'type',
          'distance',
          'moving_time',
          'elapsed_time',
          'total_elevation_gain',
          'elev_high',
          'elev_low',
          'average_speed',
          'max_speed',
          'average_heartrate',
          'max_heartrate',
          'start_latitude',
          'start_longitude'] 

data = []  # Initialise a data list
for activity in activities:
    my_dict = activity.to_dict()
    data.append([activity.id]+[my_dict.get(x) for x in my_cols])  # Sort activity data and IDs into a dictionary
    
# Add id to the beginning of the columns, used when selecting a specific activity
my_cols.insert(0,'id')

df = pd.DataFrame(data, columns=my_cols)  # Fill the dataframe with activity data

# Make all walks into hikes for consistency
df['type'] = df['type'].replace('Walk','Hike')

# Create a distance in km column
df['distance_km'] = df['distance']/1e3

# Convert dates to datetime type
df['start_date_local'] = pd.to_datetime(df['start_date_local'])

# Create a day of the week and month of the year columns
df['day_of_week'] = df['start_date_local'].dt.day_name()
df['month_of_year'] = df['start_date_local'].dt.month

# Convert timings to hours for plotting
df['elapsed_time_hr'] = df['elapsed_time'].astype(int)/3600
df['moving_time_hr'] = df['moving_time'].astype(int)/3600
```

    2021-11-01 13:54:11.120 | INFO     | stravaio:strava_oauth2:343 - serving at port 8000
    2021-11-01 13:54:13.533 | DEBUG    | stravaio:run_server_and_wait_for_token:397 - code: 53086ce8b074d740f319af9d8972ecb8b6e7d2c6
    2021-11-01 13:54:14.485 | DEBUG    | stravaio:run_server_and_wait_for_token:406 - Authorized athlete: 6103aecbe190b7468808426c036ca16cc1003f62



```python
# We can now view our dataframe containing activities and corresponding information:
df.tail(3)  
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>name</th>
      <th>start_date_local</th>
      <th>type</th>
      <th>distance</th>
      <th>moving_time</th>
      <th>elapsed_time</th>
      <th>total_elevation_gain</th>
      <th>elev_high</th>
      <th>elev_low</th>
      <th>...</th>
      <th>max_speed</th>
      <th>average_heartrate</th>
      <th>max_heartrate</th>
      <th>start_latitude</th>
      <th>start_longitude</th>
      <th>distance_km</th>
      <th>day_of_week</th>
      <th>month_of_year</th>
      <th>elapsed_time_hr</th>
      <th>moving_time_hr</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>875</th>
      <td>4397596955</td>
      <td>Morning Run</td>
      <td>2018-07-28 09:12:07</td>
      <td>Run</td>
      <td>4936.9</td>
      <td>1159</td>
      <td>1159</td>
      <td>35.7</td>
      <td>53.5</td>
      <td>27.6</td>
      <td>...</td>
      <td>5.6</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>50.93</td>
      <td>-1.41</td>
      <td>4.9369</td>
      <td>Saturday</td>
      <td>7</td>
      <td>0.321944</td>
      <td>0.321944</td>
    </tr>
    <tr>
      <th>876</th>
      <td>4397597015</td>
      <td>Lunch Run</td>
      <td>2018-07-26 12:16:34</td>
      <td>Run</td>
      <td>10579.9</td>
      <td>3183</td>
      <td>3237</td>
      <td>152.6</td>
      <td>60.6</td>
      <td>2.2</td>
      <td>...</td>
      <td>4.7</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>50.93</td>
      <td>-1.40</td>
      <td>10.5799</td>
      <td>Thursday</td>
      <td>7</td>
      <td>0.899167</td>
      <td>0.884167</td>
    </tr>
    <tr>
      <th>877</th>
      <td>4397565791</td>
      <td>Evening Run</td>
      <td>2018-07-23 18:50:51</td>
      <td>Run</td>
      <td>4913.7</td>
      <td>1090</td>
      <td>2262</td>
      <td>0.0</td>
      <td>35.6</td>
      <td>35.5</td>
      <td>...</td>
      <td>6.3</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>50.94</td>
      <td>-1.42</td>
      <td>4.9137</td>
      <td>Monday</td>
      <td>7</td>
      <td>0.628333</td>
      <td>0.302778</td>
    </tr>
  </tbody>
</table>
<p>3 rows × 21 columns</p>
</div>



## Co-ordinate extraction and Mapping 

The dataframe above is very useful for extracting key features of each activity, but it does not detail each step of the GPS tracking (i.e. each satellite 'ping' with co-ordinates). It only returns the start latitude and longitudes. We need to again access the Strava API and this time extract a complete list of all GPS pings, ensuring these data are in lat/long co-ordinates.

A limitation of the free Strava API is the ability to only make a maximum of 100 requests per 15 mins, and no more than a 1000 in 24 hours. As we are likely accessing more than 100 acitvities, we need to ensure the code waits (i.e. sleeps) for 15 mins before collecting the next batch of data. For me, this process can therefore take > 3hours.


```python
error_count = 0

for i,a in enumerate(activities):  # Loop over all activities
 
    s = client.get_activity_streams(a.id,types=['time', 'latlng', 'distance', 'altitude', # Get the key features
                                                'velocity_smooth', 'heartrate', 'cadence', 
                                                'watts', 'temp', 'moving', 'grade_smooth'])

    # Sleep for 15 mins + 1 second after extracting 90 activities (to ensure no RateLimit errors)
    
    if i!=0 and i%90 == 0:
        time.sleep(15*60 + 1)

    try:
        gps_data = s['latlng'].data  # Try to get the complete lat/long data for each activity (if it exists)
        time_data = s['time'].data  # Similarly get the time data while we are here
    except TypeError:  # TypeError can occur if these data do not exist
        error_count += 1  # Add it to the count to see how many activities had no GPS
        continue
    
    # The initial data exist in a data 'stream', so we extract the GPS data one activity at a time and store it 
    # in the df_master dataframe
    
    if i == 0:  
        df_master = pd.DataFrame( {'Run':(len(time_data)*[i]), 'time':time_data ,'gps':gps_data} )  # Create the master df
    else:
        df_iterate = pd.DataFrame( {'Run':(len(time_data)*[i]), 'time':time_data ,'gps':gps_data} )
        df_master = pd.concat([df_master, df_iterate])  # Concatenate dataframes
    
```


```python
# This is what our new dataframe looks like
df_master.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Run</th>
      <th>time</th>
      <th>gps</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>105.0</td>
      <td>[50.942063, -1.40224]</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0</td>
      <td>110.0</td>
      <td>[50.94196, -1.401999]</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0</td>
      <td>116.0</td>
      <td>[50.941857, -1.40172]</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0</td>
      <td>122.0</td>
      <td>[50.941759, -1.401468]</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0</td>
      <td>130.0</td>
      <td>[50.941598, -1.401341]</td>
    </tr>
  </tbody>
</table>
</div>




```python
# It is important to save this dataframe; so we don't have to run the above code again!
df_master.to_pickle("./df_master.pkl")

# To load again, use this line:
# df_master = pd.read_pickle("./df_master.pkl")
```


```python

```

![alt text](Soton_med.png.png "Southampton View 1")


```python

```


![alt text](Soton_small.png.png "Southampton View 2")

![alt text](Soton_large.png.png "Southampton View 3")


```python
# (Optional)
# I here define a filter function with set latitude/longitude co-ordinates to only select activities in a set boundary

def myFilterFunc(coords: list) -> bool:
    
    lat, lon = coords
    
    lat_in_range = 50.8 < lat < 51.4  # Choose some co-ordinates
    lon_in_range = -1.6 < lon < -1.2
        
    return (lat_in_range and lon_in_range)

df_filter = df_master[df_master['gps'].map(myFilterFunc)]  # My new, filtered, dataframe
```

## Folium Heatmap

We can also create a heatmap with folium, using a plugin.


```python
from folium.plugins import HeatMap
```


```python

```

![alt text](Heatmap.png.png "Heatmap view")


# Race Profile Analysis

Let's now consider a single activity (here, the Ryde 10 mile race of February 2020) and break down the GPS data (which is stored in a gpx file). This will allow us the do some insightful projections and visualisations with our data.


```python
%matplotlib inline
```


```python
import glob
from datetime import timezone

import numpy as np
import matplotlib
import matplotlib.pyplot as plt
from matplotlib import rc
from mpl_toolkits.mplot3d import Axes3D
from matplotlib.colors import LightSource

import gpxpy

matplotlib.rcParams.update({'font.size':14})
```

## GPX Data


```python
gpx_file = open('./Ryde_10m.gpx', 'r')  # Select a local gpx file of my chosen race
gpx = gpxpy.parse(gpx_file)  # Load in the gpx data with the gpxpy library

data = gpx.tracks[0].segments[0].points
df = pd.DataFrame(columns=['lat','long','elev','time'])  # Create a dataframe of latitude, longitude, elevation 
                                                         # and time, as recorded in the gpx file
    
# And fill the dataframe:

for i,val in enumerate(data):
    
    lat = val.latitude  
    long = val.longitude 
    elev = val.elevation  
    time = val.time 

    df = df.append( {'lat': lat, 'long' : long, 'elev' : elev, 'time' : time}, ignore_index=True )
```


```python
# Our data now look like this:
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>lat</th>
      <th>long</th>
      <th>elev</th>
      <th>time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>50.729605</td>
      <td>-1.147696</td>
      <td>3.2</td>
      <td>2020-02-02 10:59:29+00:00</td>
    </tr>
    <tr>
      <th>1</th>
      <td>50.729610</td>
      <td>-1.147729</td>
      <td>3.2</td>
      <td>2020-02-02 10:59:30+00:00</td>
    </tr>
    <tr>
      <th>2</th>
      <td>50.729629</td>
      <td>-1.147823</td>
      <td>3.1</td>
      <td>2020-02-02 10:59:32+00:00</td>
    </tr>
    <tr>
      <th>3</th>
      <td>50.729648</td>
      <td>-1.147931</td>
      <td>3.1</td>
      <td>2020-02-02 10:59:34+00:00</td>
    </tr>
    <tr>
      <th>4</th>
      <td>50.729656</td>
      <td>-1.147990</td>
      <td>3.1</td>
      <td>2020-02-02 10:59:35+00:00</td>
    </tr>
  </tbody>
</table>
</div>



### Race Map (with elevation overlay)


```python
# To create an elecation overlay, a precision level must be assigned (here, it is every 7 bins)

precision = 7

diff_elev = []

for i,val in enumerate(df['elev']):  # Loop over elevation data
    
    if i==0:
        start_elev = val
        diff_elev.append(0)
    if i%precision==0 and i!=0:  # When we reach the precision limit
        stop_elev = val
        
        diff = stop_elev - start_elev  # Calculate elevation change
        diff_elev.append(diff)
        
        start_elev = val  # Reset start elevation

        
long_diff_elev = []        

for i in diff_elev:
    for j in range(precision): 
        long_diff_elev.append(i) # Fill elevation change list       
```


```python
# Plot these gpx data on a 2d plot, overlaying elevation information

fig, ax = plt.subplots()

df.plot('long', 
        'lat', 
        c=long_diff_elev, 
        colormap='PuOr_r', 
        colorbar=True,
        s=40, 
        kind='scatter', 
        figsize=(8.5,8), 
        vmax=11, vmin=-11, 
        title='Ryde 10m',
        ax=ax)

ax.set_facecolor("#A2EA83")  # Set background to green

plt.text(x=-1.081, y=50.705, s='Elevation change (m)', rotation=270, fontsize=18)
```




    Text(-1.081, 50.705, 'Elevation change (m)')




    
![png](output_32_1.png)
    


## Lidar (Light and Ranging) Data

The Ryde 10 mile race is located on the Isle of Wight (IoW), in the UK. The Envrionmental Agency in the UK have published "The LIDAR Composite DTM (Digital Terrain Model)", which is "a raster elevation model covering ~75% of England at 2m spatial resolution", which is openly accessible: https://data.gov.uk/dataset/002d24f0-0056-4176-b55e-171ba7f0e0d5/lidar-composite-dtm-2017-2m 
***
I extracted the terrain data corresponding to the IoW and now process that to project this onto a new map.


```python
%matplotlib notebook
import matplotlib.pyplot as plt
```


```python
super_geo = np.zeros((5000,5000))  # Create a 2d (5000 x 5000) grid  
    
for X in range(10):  # Loop over Lidar data files
    x = 55+X
    for Y in range(10):
        y = 85+Y  
        try:
            ascii_file = "./All_lidar/sz{0}{1}_DSM_2M.asc".format(x,y)  # Take each ascii file
            
            geo = np.loadtxt(ascii_file, skiprows=6)  # Read in the Lidar data

            super_geo[500*(9-Y):(9-Y+1)*500,  500*X:500*(X+1)] = geo  # Update the 2d grid
                    
        except OSError:  # If the Lidar file isn't defined, continue to next
            continue

        super_geo[super_geo==-9999] = 0  # Set any 'error values = -9999' to be at zero elevation
  
```


```python

```


```python
# Now let's load in the gpx data (latitudes, longitudes and elevation) 
xs = df['long'] 
ys = df['lat'] 
zs = df['elev'] 
```


```python
%matplotlib inline
import matplotlib.pyplot as plt
```


```python
# Project race onto 2d Lidar graphic: 

plt.figure(figsize=(12,8))

plt.imshow(super_geo, extent=[-1.223054,-1.081792,  50.662115,50.752795])  # Lidar projection using imshow

cbar = plt.colorbar()
cbar.set_label('Elevation(m)', rotation=270, fontsize=14)

plt.xlabel('Long', fontsize=14)
plt.ylabel('Lat', fontsize=14)

plt.plot(xs, ys, linewidth=2, color='orange')  # Plot over race map

# You can clearly see where the race covers both in-land hills and the low-level waterside
```




    [<matplotlib.lines.Line2D at 0x11ab9ea60>]




    
![png](output_40_1.png)
    



```python
%matplotlib notebook
import matplotlib.pyplot as plt
```


```python

```


```python

```
