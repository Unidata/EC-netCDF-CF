# DSG Options for realtime station data

## Classic Model-compatible

### Current DSG

- One dimension for observations
- Every array has the same size
  - Repeats values for spatial metadata
  - Simple
- Need to scan entire file to find all observations for a particular station
- In netCDF3, wastes space
- Compression makes this less worrisome (?)

```
netcdf stations {
dimensions:
  obs = UNLIMITED ;      // currently 6
variables:
  float lat(obs) ;
  float lon(obs) ;
  string stid(obs) ;
  double time(obs) ;
  float temperature(obs) ;
data:

 lat = 39.9, 40, 35.25, 39.9, 35.25, 39.9 ;
 lon = -104.9, -105, -97.1, -104.9, -97.1, -104.9 ;
 stid = "KDEN", "KBOU", "KOKC", "KDEN", "KOKC", "KDEN" ;
 time = 7776000, 7776000, 7776000, 7777800, 7777800, 7778100 ;
 temperature = 15, 16, 25, 15.5, 25.2, 15.6 ;
}
```

### DSG Classic Ragged Array

- One dimension for observations, One for stations
- Extra observation-length array for for index mapping observation to station
  - Less extra space usage
  - More complicated
- Need to scan entire file to find all observations for a particular station
- Less redundant information
- Not clear the complexity (two array lookups) worth it with netCDF4

```
netcdf stations {
dimensions:
  obs = UNLIMITED ;      // currently 6
  station = UNLIMITED ;      // currently 3
variables:
  float lat(station) ;
  float lon(station) ;
  string stid(station) ;
  int index(obs) ;
  double time(obs) ;
  float temperature(obs) ;
data:

 lat = 39.9, 40, 35.25 ;
 lon = -104.9, -105, -97.1 ;
 stid = "KDEN", "KBOU", "KOKC" ;
 index = 0, 1, 2, 0, 2, 0 ;
 time = 7776000, 7776000, 7776000, 7777800, 7777800, 7778100 ;
 temperature = 15, 16, 25, 15.5, 25.2, 15.6 ;
}
```

## Enhanced Model

### Compound Data Type, Unlimited ob dimension
- One dimension for observations
- Compound data type used to keep all values for a particular observation
- Better data locality may improve I/O, compression
- Dealing with compound data type may be complex for client

```
netcdf stations {
types:
  compound obs_t {
    float lat ;
    float lon ;
    string stid ;
    double time ;
    float temperature ;
  }
dimensions:
  obs = UNLIMITED ;      // currently 6
variables:
  obs_t obs(obs) ;

data:

 obs = {39.9, -104.9, "KDEN", 7776000, 15},
       {40, -105, "KBOU", 7776000, 16},
       {35.25, -97.1, "KOKC", 7776000, 25},
       {39.9, -104.9, "KDEN", 7777800, 15.5},
       {35.25, -97.1, "KOKC", 7777800, 25.2},
       {39.9, -104.9, "KDEN", 7778100, 15.6} ;
}
```

### VLen Data Type, Unlimited Station Dimension
- One dimension for observations
- VLen data type used to handle ragged arrays
- Finding data for a station only requires finding a single index
- All data for a particular variable stored together
- Same as classic model linear array, but exploiting VLen to rationalize per-station storage

```
netcdf stations {
  types:
  float (*) variable_time_float ;
  double (*) variable_time_double ;
dimensions:
  station = UNLIMITED ;      // currently 3
variables:
  float lat(station) ;
  float lon(station) ;
  string stid(station) ;
  variable_time_double time(station) ;
  variable_time_float temperature(station) ;
data:

 lat = 39.9, 40, 35.25 ;
 lon = -104.9, -105, -97.1 ;
 stid = "KDEN", "KBOU", "KOKC" ;
 time = {7776000, 7777800, 7778100}, {7776000}, {7776000, 7777800} ;
 temperature = {15, 15.5, 15.6}, {16}, {25, 25.2} ;
}
```

### Compound Vlen Data Type nested within Compound Data Type, Unlimited Station Dim
- Most complex
- Not clear all clients can even handle this
- Puts all information for a station together
- Slicing across stations is hard

### Group-per-station
- All station metadata stored at global scope
- Each station has its own group
  - Eliminates ragged arrays, VLEN, and compound data types
  - Client must scan all groups to get data for all stations
  - No need for all stations to have same variables
- Can clients handle ~6000 groups?
- Locality probably same as using compound data type

```
netcdf stations {

dimensions:
  station = UNLIMITED ;      // currently 3
variables:
  float lat(station) ;
  float lon(station) ;
  string stid(station) ;
data:

 lat = 39.9, 40, 35.25 ;
 lon = -104.9, -105, -97.1 ;
 stid = "KDEN", "KBOU", "KOKC" ;

group: KDEN {
  dimensions:
    obs = UNLIMITED ;      // currently 3
  variables:
    int index ;
    double time(obs) ;
    float temperature(obs) ;
  data:
   index = 0 ;
   time = 7776000, 7777800, 7778100 ;
   temperature = 15, 15.5, 15.6 ;
  } // group KDEN

group: KBOU {
  dimensions:
    obs = UNLIMITED ;      // currently 1
  variables:
    int index ;
    double time(obs) ;
    float temperature(obs) ;
  data:
   index = 1 ;
   time = 7776000 ;
   temperature = 16 ;
  } // group KBOU

group: KOKC {
  dimensions:
    obs = UNLIMITED ;      // currently 2
  variables:
    int index ;
    double time(obs) ;
    float temperature(obs) ;
  data:
   index = 2 ;
   time = 7776000, 7777800 ;
   temperature = 25, 25.2 ;
  } // group KOKC
}
```
